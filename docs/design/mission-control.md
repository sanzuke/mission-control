# Rancangan Mission Control

> Status: draft v0.3 · 2026-10-11 · dokumen living (edit di tempat, riwayat lewat `git log -p`)
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
| `omp` | **Oh My Pi** (`omp`, fork Pi, [can1357/oh-my-pi](https://github.com/can1357/oh-my-pi)): (a) monitor transkrip JSONL di `~/.omp/agent/sessions/`; (b) kendali lewat `omp --mode rpc` (JSONL via stdio) | flag RPC persis **belum diverifikasi** (dokumen Pi upstream: `--mode rpc`; wrapper pihak ketiga memakai `rpc-ui`); cek `omp --help` sebelum fase 5 |
| `codex` | transkrip sesi Codex CLI + proses | seperti claude-code, format berbeda |
| `acp` | Agent Client Protocol via stdio | satu adapter untuk banyak agent |
| `openai-compat` | loop tool-use sendiri via HTTP | ditunda |

Kontrak diuji dengan rekaman event (golden files) per provider karena format tiap CLI berubah antarversi.

## 6. Dunia 3D

- **Scene**: kampus/kantor toon-shaded; **ruangan per proyek** (dari blueprint JSON) dengan meja kerja dan meja rapat. Lingkungan dapat diganti (kantor malam, taman, dst.).
- **Kamera**: 3 level zoom — overview → fokus ruangan → close-up karakter. Aksi: klik karakter = Inspector; drag karakter ke ruangan lain = pindahkan agent; gelembung aktivitas di atas kepala.
- **Karakter**: bot dengan kepribadian/peran (Worker, Thinker, Coder, Designer, Comms…), aksesori per peran. Mulai prosedural; dukungan GLB belakangan.
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

1. **Bangun baru atau fork/kontribusi ke CrewHub?** Fork = cepat tetapi terikat AGPL dan arah proyek mereka. Rekomendasi: bangun baru, bila tujuan Anda berbeda (mis. kanban+worktree+approval sebagai inti).
2. **OMP** = Oh My Pi (dikonfirmasi). Perlu: versi `omp` yang dipakai, dan output `omp --help` / contoh satu file sesi di `~/.omp/agent/sessions/` agar parser tidak menebak.
3. Di mana ini dijalankan: laptop lokal, VPS/server, atau keduanya? (menentukan apakah modul metrics/tunnel dibutuhkan)
4. Gaya karakter: robot (seperti referensi), orang/chibi, atau hewan? Pakai aset sendiri atau kami buatkan?
5. Provider mana yang paling dulu setelah Claude Code: OMP, Codex, atau yang lain?
