# Rancangan Mission Control

> Status: draft v0.4 · 2026-10-11 · dokumen living (edit di tempat, riwayat lewat `git log -p`)
> Referensi (bukan spesifikasi mengikat):
> - Google Doc "Mission Control": ternyata prompt untuk muse.ai. Diambil ide fiturnya, bukan batasan tekniknya.
> - [EKINSOL-DEV/crewhub](https://github.com/EKINSOL-DEV/crewhub) (AGPL-3.0): dipelajari arsitekturnya (lihat bagian 11). **Tidak ada kode yang disalin.**

## 1. Tujuan

Dashboard untuk memantau dan mengarahkan banyak AI coding agent. Agent tampil sebagai **karakter 3D** di ruang kerja virtual, tugasnya dikelola di **kanban**.

1. **Claude Code** = integrasi paling dalam (monitor, kendali, approval).
2. **OMP (Oh My Pi)** dan provider lain (Codex CLI, OpenClaw, Gemini CLI, API OpenAI-compatible) lewat satu antarmuka koneksi.
3. **Kejujuran data**: status yang tampil harus berasal dari sumber nyata; kalau tak tersedia tampilkan "tidak tersedia", bukan angka palsu.

Bukan tujuan v1: editor kode di UI, multi-user/SaaS.

## 2. Arsitektur

```
┌──────────── Web UI (React + TS + react-three-fiber) ────────────┐
│ World 3D │ Kanban │ Agents │ Inspector/Chat │ Log │ Metrics     │
└──────▲───────────────────────────────┬──────────────────────────┘
       │ SSE (event stream)            │ REST (perintah)
┌──────┴───────────────────────────────▼──────────────────────────┐
│ Daemon (Node + TS, Fastify)                                      │
│  ConnectionManager ─ EventBus ─ TaskService ─ ApprovalBroker     │
│  WorktreeManager ─ Scheduler ─ SQLite (tasks, agents, events)    │
└──┬───────────────┬────────────────┬───────────────┬─────────────┘
   │               │                │               │
 ClaudeCode      OMP            Codex/ACP        OpenAI-compat
 (transkrip JSONL + hooks + proses `claude`)
```

Semua provider mengimplementasikan satu `AgentConnection` dan menghasilkan `MissionEvent` ternormalisasi. UI tidak tahu provider mana yang dipakai; ia hanya membaca `capabilities`.

### Stack
| Lapisan | Pilihan | Alasan |
|---|---|---|
| Daemon | Node + TypeScript, Fastify | satu bahasa dengan UI; Claude Agent SDK berbasis TS |
| Real-time | SSE (+ REST untuk perintah) | cukup untuk arus satu arah, lebih sederhana dari WebSocket |
| Storage | SQLite + tabel event append-only | lokal, tanpa server |
| UI | React, Vite, Tailwind, Zustand | |
| 3D | three.js + react-three-fiber + drei | toon-shaded, karakter GLB/prosedural |
| Packaging | `npx`/Docker; opsional Tauri | |

Lihat ADR-0001. Mode Docker hanya bisa monitor (tak bisa menjalankan `claude`) — didokumentasikan jelas.

## 3. Model data

- **Agent**: `id, name, provider, model, persona, avatar, roomId, settings`.
- **Session**: satu run provider (`providerSessionId`, status, activity, biaya/token).
- **Task/Kartu**: `id, title, description, column, priority, labels, assignee, deps, branch, worktree, prUrl, acceptance`.
- **Room**: dari **blueprint JSON** (tata letak, prop, meja kerja) — data-driven.
- **MissionEvent** (append-only): `session.started | activity.changed | message | tool.start | tool.end | permission.requested | permission.resolved | subagent.started | subagent.ended | usage | error | session.ended`.
- **AgentActivity**: `idle | thinking | tool_use | responding | waiting_input | waiting_permission | error`.

## 4. Integrasi Claude Code (utama)

Tiga sumber dipadukan; hooks memberi kepastian, transkrip memberi kelengkapan:

1. **Watcher transkrip** — pantau `~/.claude/projects/*/*.jsonl` (scan file baru ±1 dtk, deteksi perubahan lewat `stat`). Parser bertipe untuk record `assistant | user | system | progress | summary | file-history-snapshot | queue-operation`. Rekaman nyata punya `progress` bertipe `bash_progress` dan `hook_progress`; `summary` dipakai untuk judul sesi. Record tak dikenal **dicatat dan diabaikan**, tidak membuat crash.
2. **Hooks** (`SessionStart, PreToolUse, PostToolUse, Notification/PermissionRequest, SubagentStop, Stop`) → `POST /hooks/claude`. Ini memberi sinyal pasti untuk `waiting_permission` dan akhir giliran, sehingga tidak perlu menebak dari timeout. Hooks bersifat opsional: tanpa hooks, fallback ke heuristik transkrip (tool_use tanpa hasil ±7 dtk ⇒ menunggu izin; teks tanpa lanjutan ±5 dtk ⇒ menunggu input).
3. **Kendali** — Claude Agent SDK / spawn `claude -p --output-format stream-json --resume` dengan `cwd` = git worktree kartu; `canUseTool` terhubung ke ApprovalBroker (approve/deny dari UI); interrupt.

Tambahan: **MCP server kanban** agar Claude bisa `list_tasks / claim_task / update_task / add_subtask`; **Handoff** "Lanjutkan di…" (Terminal/VS Code dengan `claude --resume <id>`); subagent muncul sebagai karakter kecil di dekat induknya; judul aktivitas yang mudah dibaca ("Reading config.py", "Running: npm test") dari input tool.

## 5. Lapisan provider

```ts
interface AgentConnection {
  id: string;
  capabilities: { observe; run; interrupt; approvals; resume; subagents; usage: boolean };
  discover(): Promise<SessionInfo[]>;
  history(sessionId): Promise<HistoryMessage[]>;
  start(opts: { cwd; prompt; model?; agentId }): Promise<Session>;
  send(session, text): Promise<void>;
  interrupt(session): Promise<void>;
  events(): AsyncIterable<MissionEvent>;
}
```
| Adapter | Cara | Catatan |
|---|---|---|
| `claude-code` | transkrip + hooks + proses/SDK | fitur penuh |
| `omp` | Oh My Pi v18.8.9 (`omp`, [can1357/oh-my-pi](https://github.com/can1357/oh-my-pi)). Tiga jalur, lihat 5.1 | `omp acp` adalah jalur utama untuk kendali; skema event masih menunggu sampel |
| `codex` | transkrip sesi Codex CLI + proses | seperti claude-code, format berbeda |
| `acp` | Agent Client Protocol via stdio | satu adapter untuk banyak agent |
| `openai-compat` | loop tool-use sendiri via HTTP | ditunda |

### 5.1 Adapter OMP (Oh My Pi)

Dari `omp --help` (v18.8.9) tersedia hal-hal berikut. Yang bertanda ✔ langsung terbaca di help; yang bertanda ? adalah inferensi yang harus diuji.

| Kebutuhan | Cara di OMP | Status |
|---|---|---|
| Kendali dua arah | `omp acp` — server **Agent Client Protocol** via stdio (JSON-RPC). Dipakai bersama adapter `acp` generik | ✔ ada; ? cakupan event & permission |
| Alternatif kendali | `omp --mode rpc` / `rpc-ui` (+ `--no-ui` untuk headless) | ✔ ada; ? skema pesan |
| Satu kali jalan | `omp -p "…" --mode json` (keluaran JSON) | ✔ ada; ? apakah streaming per event |
| Pantau sesi | berkas JSONL di `~/.omp/agent/sessions/--<path>--/session.jsonl` (bisa digeser `PI_CODING_AGENT_DIR` / `--session-dir`) | ✔ format v3 terverifikasi dari sampel: `intent` per tool, `stopReason`, biaya per pesan |
| Izin tool | `--approval-mode always-ask\|write\|yolo`, `--auto-approve`; di `rpc-ui` "tool UI" tetap aktif | ✔ flag ada; ? bagaimana permintaan izin dikirim lewat ACP/RPC |
| Isolasi | `--profile <nama>` (auth, sesi, settings terpisah), `--cwd`, `--add-dir`, `--no-session` | ✔ |
| Worktree | `omp worktree` (add/list/clear) — bisa dipakai WorktreeManager atau cukup `git worktree` kita sendiri | ✔ ada |
| Hook/ekstensi | `--hook <file>` / `-e <file>` memuat ekstensi → bisa mendorong event ke Mission Control (setara hooks Claude) | ✔ ada; ? API ekstensi |
| Subagent | tool `task` meluncurkan subagent paralel; `omp anonymize` menyebut "session and its subagents" → subagent punya berkas sendiri | ✔ tersirat |
| Batas/biaya | `omp usage` (limit provider tiap akun), `omp stats`; `--max-time` | ✔ → menu Token/Usage |
| Proses latar | `omp ps` (daftar & kendali proses daemon-supervised) | ✔ |
| Mode/peran model | `--model`, `--smol`, `--slow`, `--plan`; `--goal`, `--plan-yolo` | ✔ → tampil di Inspector |
| Impor lintas-agent | `--from-claude`, `--from-codex` | ✔ bisa untuk "handoff" Claude → OMP |
| Kolaborasi | `omp collab` (host session lokal), `omp stream` | ? tidak dipakai v1 |

Urutan implementasi: (1) **observe** lewat watcher berkas sesi; (2) **run/interrupt** lewat `omp acp`; (3) **approval** setelah skema izin ACP diverifikasi; fallback `--mode rpc`. Format berkas sesi sudah dianalisis dari sampel nyata di [omp-session-format.md](omp-session-format.md) (observe siap dirancang; sisanya masih menunggu sampel). Seluruh asumsi ? ditutup dengan "golden file" dari sampel nyata sebelum Fase 5.

Kontrak diuji dengan rekaman event (golden files) per provider karena format tiap CLI berubah antarversi.

## 5.2 Penerapan lokal + VPS (dua-duanya)

Watcher transkrip, hooks, dan proses agent berjalan **di mesin tempat agent hidup**, sedangkan dashboard bisa di mesin lain. Maka dibedakan dua peran:

```
Laptop A ── Collector ──┐                      ┌── Browser (laptop / HP)
Laptop B ── Collector ──┼──► Daemon (VPS atau lokal) ──┤
VPS agent ─ Collector ──┘   (SQLite, SSE, Kanban)      └── (lewat tunnel/TLS)
```

- **Collector** = proses kecil di sisi agent: menjalankan adapter (watcher transkrip, hooks receiver lokal, spawn `claude`/`omp`), lalu mengirim `MissionEvent` ke daemon dan menerima perintah (start/interrupt/approve).
- **Daemon** = pusat: event bus, SQLite, kanban, SSE ke UI. Pada mode lokal, collector dan daemon berjalan satu proses (`mission-control` tanpa argumen).
- Koneksi collector → daemon: keluar dari collector (outbound WebSocket/HTTPS) sehingga laptop tidak perlu port terbuka; autentikasi dengan token per-collector yang bisa dicabut.
- Setiap `Agent` dan `Session` membawa `hostId`; worktree dan eksekusi `claude`/`omp` terjadi di host pemilik repo.
- Konsekuensi: sinkronisasi repo/worktree antar-host **bukan** tanggung jawab kita; kartu kanban menunjuk ke `hostId` + path repo.
- Metrics host (CPU/memori/disk) dikirim collector, bukan dibaca daemon.

## 6. Dunia 3D

- **Scene**: kampus/kantor toon-shaded; **ruangan per proyek** (dari blueprint JSON) dengan meja kerja dan meja rapat. Lingkungan dapat diganti (kantor malam, taman, dst.).
- **Kamera**: 3 level zoom — overview → fokus ruangan → close-up karakter. Aksi: klik karakter = Inspector; drag karakter ke ruangan lain = pindahkan agent; gelembung aktivitas di atas kepala.
- **Karakter (campuran gaya)**: tiap agent memilih `bodyType` = `robot | human | animal` + palet warna + aksesori per peran (Coder, Designer, Researcher, QA, PM…). Satu rig/skeleton standar dan satu set klip animasi dipakai semua tipe, sehingga menambah gaya baru = menambah model, bukan menulis ulang animasi. Fase 2 memakai robot prosedural; human/animal ditambahkan sebagai GLB (aset sendiri atau di-generate lalu di-rig ke skeleton standar). Provider ikut terbaca visual lewat lencana kecil di karakter (Claude/OMP/dll.).
- **Gerak**: jalan antar-ruangan dengan pathfinding A\*, tolak-menolak antar bot supaya tidak menumpuk, spawn sadar-grid.
- **Mapping aktivitas → animasi**:
  | Activity | Visual |
  |---|---|
  | idle | berdiri/berkeliling di area santai |
  | thinking | tangan di dagu, gelembung "…" |
  | tool_use (Read/Grep) | menatap layar/dokumen |
  | tool_use (Edit/Write) | duduk di meja, mengetik |
  | tool_use (Bash) | menekan tombol, lampu meja berkedip |
  | responding | bicara, gelembung teks |
  | waiting_permission | angkat tangan + ikon ❗ (klik → approve/deny) |
  | error | kaget, lampu merah |
  | selesai | cheer hop, kartu → Review |
- **Performa/aksesibilitas**: low-poly, instancing, target 60 fps laptop biasa; mode **Zen/2D** (daftar agent + chat) bila WebGL lemah atau `prefers-reduced-motion`; mobile: viewport 3D mengambang kecil + panel responsif.

## 7. Kanban & task management

Kolom: `Backlog → Ready → In Progress → Review → Done` (+ `Blocked`).

- **Run with Agent**: assign + start → worktree & branch `mc/<id>-<slug>`, session dijalankan, kartu → In Progress.
- Agent selesai → **Review** (bukan Done); manusia memilih *Terima* atau *Minta revisi* (komentar kembali ke agent sebagai prompt).
- `waiting_permission/input` → label `needs-you`; kegagalan/stale → `Blocked` dengan alasan.
- Dependensi antar-kartu, WIP limit per agent, batas anggaran per kartu, filter per agent/label.
- Agent bisa menambah subtask lewat MCP; action item dari **rapat agent** (lihat bagian 8) bisa dijadikan kartu.
- Opsional: sinkron dua arah dengan GitHub Issues/Projects.

## 8. Fitur kolaborasi & operasional (modul opsional)

Dipilih dari ide referensi, dikerjakan setelah inti stabil:
- **Group chat & rapat terstruktur** (round-robin antar-agent → ringkasan, keputusan, action item → kartu).
- **Persona tuning** per agent (preset + sifat).
- **System/VPS metrics** (CPU, memori container dari `/proc`, disk, uptime) dan **jadwal cron** (file `.md` berfrontmatter) bila Mission Control dijalankan di server.
- **Tunnel + watchdog** untuk akses jarak jauh; lihat keamanan.
- **Mode demo** dengan agent tiruan yang diberi label jelas "DEMO" (satu-satunya tempat data simulasi diizinkan).
- **Modding**: prop/lingkungan/blueprint lewat JSON dengan ID ber-namespace.

## 9. Keamanan

Referensi (CrewHub) berangkat dari "tanpa autentikasi, khusus jaringan tepercaya". Kita tidak mengikuti default itu:
- Bind `127.0.0.1`; **token wajib** untuk REST, SSE, dan receiver hooks; akses jarak jauh hanya lewat TLS + auth.
- Perintah yang bisa dijalankan UI dibatasi (whitelist adapter); tidak ada shell bebas.
- Secret provider tidak masuk event log/UI. Transkrip dapat memuat secret: tampilkan terpotong, dan jangan kirim keluar mesin.
- Agent bekerja di worktree terisolasi; merge hanya lewat aksi manusia di kolom Review.
- Aksi berisiko selalu lewat ApprovalBroker.

## 10. Roadmap

| Fase | Isi | Selesai bila |
|---|---|---|
| 0 | Repo, daemon, SQLite, SSE, skema `MissionEvent`, mode demo | event dummy ter-stream ke UI |
| 1 | Watcher transkrip Claude Code + parser + Agent list | sesi `claude` nyata muncul & berubah status |
| 2 | World 3D v1: 1 ruangan, 3 bot, 6 animasi inti | animasi cocok dengan aktivitas nyata |
| 3 | Kanban + worktree + "Run with Agent" + Review | satu kartu end-to-end |
| 4 | Hooks + ApprovalBroker + Inspector/Chat + MCP kanban | approve dari UI |
| 5 | Adapter OMP + Codex/ACP | dua provider berjalan bersamaan |
| 6 | Rooms/blueprint, rapat agent, metrics/cron, mobile, polish | siap rilis |

## 11. Pelajaran dari CrewHub

Diambil (ide):
- Abstraksi koneksi + `ConnectionManager`; semua sumber → satu bentuk `SessionInfo`/`HistoryMessage`.
- Monitor Claude Code lewat transkrip JSONL tanpa mengubah Claude; SSE untuk push.
- Aktivitas terbaca manusia di bot/chat; blueprint ruangan JSON; mode Zen; handoff `claude --resume`; mode demo.
- Dokumen gap-analysis mereka menunjukkan jebakan: parser yang diam-diam membuang tipe record, perbedaan antara rekaman nyata dan asumsi (mis. `agent_progress` tak pernah muncul), dan mode Docker yang tak bisa menjalankan CLI.

Dibedakan:
- Autentikasi aktif secara default (lihat bagian 9).
- Hooks sebagai sinyal izin yang pasti, bukan hanya heuristik timeout.
- Kanban terikat git worktree + alur Review manusia; ApprovalBroker dari UI.
- Lisensi: CrewHub AGPL-3.0 — menyalin kode membuat turunan AGPL. Kita menulis dari nol.

## 12. Pertanyaan terbuka

Sudah dijawab: bangun baru (bukan fork CrewHub); lokal **dan** VPS; gaya karakter campuran; OMP = Oh My Pi v18.8.9.

1. **Sampel OMP tambahan** (sampel sesi sudah diterima dan dianalisis): sesi dengan `--approval-mode always-ask` (untuk izin), sesi yang memakai tool `task`/`edit` (subagent dan edit), serta keluaran `omp acp` / `omp -p --mode json` untuk satu prompt dengan satu tool. Daftar lengkap di bagian 6 `omp-session-format.md`. Redaksi dulu dengan `omp anonymize`.
2. Jalur OMP mana yang Anda pakai sehari-hari: sesi interaktif di terminal (→ cukup observe) atau Anda ingin Mission Control yang **menjalankan** OMP (→ acp/rpc)?
3. Apakah Claude Code dan OMP sering dipakai di repo yang sama (menentukan aturan satu-worktree-per-kartu dan pemilihan agent per kartu)?
4. Siapa saja yang mengakses dashboard dari VPS (hanya Anda, atau tim)? Menentukan apakah cukup token tunggal atau perlu akun.
5. Aset karakter: pakai model gratis/lisensi terbuka, atau ada yang ingin dibuat khusus?
