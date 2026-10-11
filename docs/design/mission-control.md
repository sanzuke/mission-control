# Rancangan Mission Control

> Status: draft v0.5 · 2026-10-11 · dokumen living (edit di tempat, riwayat lewat `git log -p`)
> Referensi (bukan spesifikasi mengikat):
> - Google Doc "Mission Control": ternyata prompt untuk muse.ai. Diambil ide fiturnya, bukan batasan tekniknya.
> - [EKINSOL-DEV/crewhub](https://github.com/EKINSOL-DEV/crewhub) (AGPL-3.0): dipelajari arsitekturnya (lihat bagian 11). **Tidak ada kode yang disalin.**

## 1. Tujuan

Tiga lapis, berurutan dari yang paling penting:

1. **Monitoring agent** — lihat semua agent (Claude Code, OMP, dst.) di semua host: sedang apa, di proyek mana, sudah memakai berapa. Sumber data harus nyata; kalau tak tersedia tampilkan "tidak tersedia", bukan angka palsu.
2. **Assign agent ke proyek** — agent adalah identitas persisten yang bisa ditugaskan ke satu atau beberapa proyek.
3. **Task management per proyek** — setiap proyek punya papan kanban sendiri; kartu bisa ditugaskan ke agent yang tergabung di proyek itu.

Karakter 3D adalah cara menampilkan (1) dan (2): **satu ruangan = satu proyek**, karakter masuk ke ruangan saat di-assign. Integrasi Claude Code paling dalam; OMP dan provider lain lewat satu antarmuka koneksi.

**Monitoring dulu, kendali belakangan.** v1 cukup *mengamati* dan *mengorganisir*. Menjalankan agent dari kartu ("Run with Agent"), approval dari UI, dan chat adalah tahap berikutnya.

Bukan tujuan v1: editor kode di UI, multi-user/SaaS, menjalankan agent dari dashboard.

## 1.1 Konsep inti

```
Host ──< Session >── Agent ──< Assignment >── Project ──< Task (kartu)
 (mesin)  (run nyata)  (identitas)   (niat)      (repo)       (kanban proyek)
```

- **Agent** = identitas persisten (mis. "Claude – laptop A", "OMP – VPS"), bukan satu sesi. Sesi datang dan pergi; agent tetap.
- **Session** = satu run yang terdeteksi dari transkrip/hook. Dipasangkan ke agent dengan aturan: `provider + host` (otomatis) atau penamaan manual.
- **Project** = satu repo/folder (`name`, `repoPath` per host, `remoteUrl`). Dikenali dari `cwd` sesi (OMP: field `cwd` di header; Claude: folder di `~/.claude/projects/`).
- **Assignment** = *niat*: agent X ditugaskan ke proyek P (boleh banyak-ke-banyak, dengan peran opsional: coder, reviewer…).
- **Observasi = kenyataan**: proyek tempat sesi benar-benar berjalan ditentukan `cwd`. Jika berbeda dari assignment, tampilkan **"bekerja di luar proyek"** (bukan dikoreksi diam-diam) dan tawarkan *assign sekarang*.
- **Task** = milik tepat satu proyek. `assignee` hanya boleh agent yang ter-assign ke proyek itu.
- **Tautan task ↔ sesi**: tidak ditebak. Cara yang sah: (a) agent mengklaim kartu lewat MCP, (b) pengguna menautkan manual, (c) saran otomatis ("agent X punya 1 kartu In Progress di proyek ini dan sesinya baru mulai") yang harus dikonfirmasi. Sesi tanpa tautan tetap tampil di proyeknya sebagai "aktivitas tanpa kartu".

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

- **Host**: `id, name, kind (laptop|vps), lastSeen, collectorVersion`.
- **Project**: `id, name, slug, color, description`, plus **`ProjectPath`** per host: `(projectId, hostId, repoPath, remoteUrl?)`. Proyek dikenali dengan mencocokkan `cwd` sesi ke `repoPath` (akar git, bukan sub-folder); `cwd` yang belum dikenal masuk **Inbox "proyek belum terdaftar"** untuk di-assign/dibuat.
- **Agent**: `id, name, provider, model?, hostId?, persona, avatar{bodyType,palette,accessory}, matchRule`.
- **Assignment**: `(agentId, projectId, role?, since)`.
- **Session**: `id, agentId, hostId, providerSessionId, cwd, projectId (turunan dari cwd), title, startedAt, lastEventAt, activity, usage{tokens,cost}`.
- **Task (kartu)**: `id, projectId, title, description, column, priority, labels, assigneeAgentId?, linkedSessionIds[], deps[], dueAt?, createdBy`. Kolom default per proyek: `Backlog → Ready → In Progress → Review → Done` (+ `Blocked`), bisa disesuaikan per proyek.
- **Room**: tata letak 3D per proyek dari blueprint JSON (1 proyek = 1 ruangan).
- **MissionEvent** (append-only): `session.started | session.titled | session.model | activity.changed | message | tool.start | tool.end | permission.requested | permission.resolved | subagent.started | subagent.ended | usage | error | session.ended` + `task.created | task.moved | task.assigned | assignment.changed` (jejak audit task/assignment).
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

- **Scene**: kampus/kantor toon-shaded; **satu ruangan per proyek** (blueprint JSON) dengan meja kerja per agent anggota dan papan kanban mini di dinding (jumlah kartu per kolom, data asli). Agent yang belum di-assign berada di **lobi**; **drag karakter ke ruangan = membuat assignment** (dengan konfirmasi). Agent yang sedang bekerja di luar proyek assign-nya tampil dengan penanda dan garis ke ruangan tempat ia benar-benar bekerja. Lingkungan dapat diganti (kantor malam, taman, dst.).
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

## 7. Task management per proyek

Setiap proyek punya papan sendiri (tidak ada papan global; tampilan "Semua proyek" hanyalah agregasi baca-saja).

**v1 (tanpa menjalankan agent):**
- Buat/edit/pindah/hapus kartu, drag antar kolom, prioritas, label, tenggat, dependensi.
- **Assign kartu ke agent** yang ter-assign ke proyek (dropdown hanya menampilkan anggota proyek).
- Kartu menampilkan badge status agent langsung: agent pemilik kartu sedang `tool_use`, `waiting_input`, offline, dll. (dari monitoring).
- **Aktivitas tanpa kartu**: sesi agent di proyek yang belum ditautkan ke kartu muncul di panel samping proyek, bisa diubah jadi kartu ("Buat kartu dari sesi ini") atau ditautkan ke kartu yang ada.
- Tautan kartu ↔ sesi (manual atau konfirmasi saran) membuat tab *Sesi* di kartu: judul, durasi, model, token/biaya, aktivitas terakhir.
- Gerak kolom otomatis hanya **saran**, bukan aksi diam-diam: mis. sesi tertaut selesai (`stop`) → banner "Pindahkan ke Review?". Pengguna yang memutuskan; `Done` hanya oleh manusia.
- Ringkasan per proyek: agent anggota (+ aktivitas), kartu per kolom, biaya total sesi tertaut, kartu macet (In Progress tanpa aktivitas agent > N jam).
- Opsional: impor/sinkron dengan GitHub Issues/Projects.

**Tahap berikutnya (kendali):**
- **Run with Agent**: kartu di-Start → git worktree + branch `mc/<id>-<slug>`, sesi dijalankan lewat Collector, otomatis tertaut ke kartu.
- Agent mengambil/menambah/memperbarui kartu sendiri lewat **MCP server kanban** (tool dibatasi pada proyek tempat agent ter-assign).
- Review → *Terima / Minta revisi* (komentar jadi prompt baru), batas anggaran per kartu, WIP limit per agent, action item rapat agent → kartu.

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

Urutan mengikuti prioritas: monitoring → proyek & assignment → kanban per proyek → 3D → kendali.

| Fase | Isi | Selesai bila |
|---|---|---|
| 0 | Repo, Daemon+Collector kerangka, SQLite, SSE, skema `MissionEvent`, fixture dari sampel | event dari fixture ter-stream ke UI |
| 1 | **Monitoring**: watcher + parser OMP (v3) dan Claude Code; daftar Agent & Sesi dengan aktivitas, model, token/biaya | sesi nyata dari kedua provider tampil & berubah status |
| 2 | **Proyek & assignment**: CRUD proyek, pemetaan `cwd`→proyek, Inbox proyek baru, assign agent↔proyek, penanda "bekerja di luar proyek" | agent bisa di-assign dan sesinya muncul di proyek yang benar |
| 3 | **Kanban per proyek** (v1 tanpa menjalankan agent): kartu, kolom, assign ke agent, tautan kartu↔sesi, aktivitas tanpa kartu | satu proyek dikelola penuh dari papan |
| 4 | **Dunia 3D**: ruangan per proyek, karakter, animasi dari aktivitas, drag-to-assign, mini-kanban di dinding, mode Zen/2D | tampilan 3D sinkron dengan data nyata |
| 5 | Multi-host: Collector jarak jauh (VPS), token per-collector, hooks Claude | agent dari laptop dan VPS tampil bersamaan |
| 6 | **Kendali**: Run with Agent + worktree, approval, chat, MCP kanban, adapter OMP via `acp` | kartu → sesi → Review end-to-end |
| 7 | Rapat agent, metrics/cron, mobile, polish | siap rilis |

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

Sudah dijawab: bangun baru; lokal dan VPS; karakter campuran; OMP = Oh My Pi; konsep inti = monitoring → assign agent ke proyek → kanban per proyek.

1. **Definisi "agent"**: apakah satu agent = satu kombinasi *provider + host* (mis. "Claude di laptop A"), atau Anda memberi nama sendiri (mis. "Muse", "Aris") lalu memetakan sesi ke nama itu? Rekomendasi: otomatis `provider+host`, bisa diganti nama dan digabung manual.
2. **Satu agent boleh di banyak proyek sekaligus?** Rekomendasi: ya (banyak-ke-banyak), karena sesi sebenarnya ditentukan `cwd`.
3. **Pembuatan proyek**: otomatis dari `cwd` baru (masuk Inbox) lalu Anda konfirmasi, atau selalu dibuat manual dulu?
4. **Kolom kanban**: pakai kolom default (Backlog → Ready → In Progress → Review → Done, + Blocked) atau Anda punya alur sendiri?
5. **Sinkron dengan layanan task lain** (GitHub Issues/Projects, Huly, dll.) — perlu di v1, atau kanban lokal cukup? (Di sampel OMP terlihat Anda memakai konteks proyek "Huly".)
6. **Sampel OMP tambahan** (izin `always-ask`, `task`/subagent, `edit`, `omp acp`) — lihat bagian 6 `omp-session-format.md`; hanya dibutuhkan untuk fase kendali dan subagent.
7. Aset karakter: model gratis berlisensi terbuka atau dibuat khusus?
