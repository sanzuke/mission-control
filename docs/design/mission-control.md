# Rancangan Mission Control

> Status: draft v0.7 · 2026-10-11 · dokumen living (edit di tempat, riwayat lewat `git log -p`)
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

Bukan tujuan v1: editor kode di UI, SaaS multi-tenant, menjalankan agent dari dashboard. Anggota manusia ada sejak v1, tetapi pada awalnya dikelola oleh QA (sebagai proxy) dan lewat sinyal Git/CI (lihat 7.2 dan 7.5); akun login ringan menyusul.

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
- **Tautan task ↔ sesi (banyak-ke-banyak, berjangka waktu)**: satu sesi sering mengerjakan beberapa pekerjaan berurutan (mis. memperbarui dokumen progres, lalu memperbaiki FE, lalu testing), dan satu task bisa dikerjakan beberapa sesi. Maka tautan bukan `session → 1 task`, melainkan **segmen**: `(sessionId, taskId, dari, sampai)`. Sesi punya **Fokus** (task yang sedang dikerjakan); pengguna/agent mengganti fokus (satu klik di UI atau tool MCP `set_focus`), dan setiap pergantian menutup segmen lama serta membuka yang baru. Biaya/token dan aktivitas dihitung per segmen. Tidak ditebak: sistem hanya **menyarankan** pergantian fokus (mis. `cwd`/berkas yang disentuh berubah, atau `intent` menyebut pekerjaan lain) dan menunggu konfirmasi. Sebelum ada fokus, aktivitas masuk "tanpa kartu".

## 1.2 Skenario acuan: siklus bug QA

Pemilik adalah QA (manual + automation); satu sesi agent bisa memegang beberapa peran, dan sebuah bug melewati beberapa tim/agent:

```
Testing (QA)  ──gagal──►  [Bug]  ──►  [Fix]  ──►  [Deploy]  ──►  [Retest]  ──lulus──►  [Update dokumen]
   ▲                       QA        role FE     env + build     QA, env sama                role Docs
   │                                                                │
   └───────────── gagal lagi: [Bug] baru, tertaut ke Retest (iterasi n+1) ◄┘
```

Aturan yang bisa dijalankan sistem **secara deterministik** (bukan tebakan):
- Kartu `Fix` selesai (`Done` oleh manusia) → `Deploy` otomatis berpindah dari `Blocked` ke `Ready` (dependensi terpenuhi).
- `Deploy` `Done` → `Retest` terbuka dan agent berperan QA di proyek itu diberi notifikasi.
- `Retest` punya field hasil `lulus | gagal`. **Gagal** → sistem *menyarankan* kartu `Bug` baru (terisi bukti dan tertaut `retestOf`) dan menaikkan penghitung iterasi; **lulus** → `Update dokumen` terbuka dan bug induk bisa ditutup (oleh manusia).
- Semua yang bersifat keputusan (Done, tutup bug, buat bug dari kegagalan) tetap konfirmasi manusia.

Satu `Retest` harus menunjuk **env + build/commit yang sama** dengan `Deploy` sebelumnya; kalau berbeda, UI menampilkan peringatan.

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
- **Assignment**: `(memberId, projectId, roles[], since)`; peran proyek: `qa | fe | be | docs | pm | …` (daftar bisa diubah per proyek). Satu agent boleh memegang beberapa peran. **Peran melekat pada assignment dan task, bukan pada agent.**
- **Member**: `agent | person` (polimorfik). Assignee/anggota bisa agent AI maupun manusia (tim FE campuran). `person` punya `externalIdentities{github?, gitlab?, email?, notion?}` untuk memetakan penulis MR/PR dan notifikasi ke orangnya.
- **Session**: `id, agentId, hostId, providerSessionId, cwd, projectId (turunan dari cwd), title, startedAt, lastEventAt, activity, focusTaskId?, usage{tokens,cost}`.
- **TaskLink (segmen)**: `(sessionId, taskId, fromEventId, toEventId?)` — rentang peristiwa sesi yang dikerjakan untuk satu task; biaya/aktivitas dihitung per segmen.
- **Task (kartu)**: `id, projectId, kind, title, description, column, priority, labels, assigneeMemberId?, assigneeRole?, deps[], dueAt?, env?, build?, result? (lulus|gagal, untuk Retest), iteration, evidence[], testIds[], fingerprint?, externalRefs[], createdBy`. Kolom default per proyek: `Backlog → Ready → In Progress → Review → Done` (+ `Blocked`), bisa disesuaikan per proyek. `kind` ∈ `task | bug | fix | deploy | retest | docs` (bisa ditambah). `assigneeRole` tanpa `assigneeMemberId` = **antrean peran** (diambil anggota berperan itu).
- **TaskRelation**: `(fromTaskId, toTaskId, type)` dengan `type` ∈ `blocks | parent | foundBy | retestOf | duplicates`.
- **WorkflowTemplate** (per proyek): rantai kartu dengan peran, kolom awal, dan aturan buka-blokir; template bawaan **"Siklus bug"** (Bug → Fix → Deploy → Retest → Update dokumen).
- **TestRun**: `projectId, source (playwright|junit|manual), env, build (commit SHA), pipelineUrl?, startedAt, results[]`. **TestResult**: `testId (file + judul), status (passed|failed|flaky|skipped), error, durationMs, retries, attachments[] (screenshot/video/trace sebagai tautan)`.
- **Fingerprint bug**: hash dari `testId + tanda tangan galat` (pesan galat yang dinormalkan: angka/ID/timestamp dibuang); kegagalan dengan fingerprint sama memperbarui bug yang ada (iterasi/bukti baru), bukan membuat duplikat.
- **Integration**: `type (notion|gdocs|github|gitlab|…), config, secretRef` — kredensial tidak disimpan di tabel, hanya referensi ke penyimpanan rahasia. **ProjectIntegration**: tautan integrasi ke proyek dengan target (database Notion, dokumen Google, repo, jalur hasil tes, nama job deploy per `env`).
- **JournalEntry**: `projectId, date, author, body, generated?, published[]{target, remoteId, publishedAt}` — jurnal progres harian per proyek (lihat 7.3).
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
- Tautan kartu ↔ sesi berupa **segmen** (lihat 1.1): kartu punya tab *Sesi* dengan daftar segmen (judul sesi, rentang waktu, model, token/biaya, aktivitas terakhir); sesi punya penanda **Fokus** yang bisa diganti cepat.
- Gerak kolom otomatis hanya **saran**, bukan aksi diam-diam: mis. sesi tertaut selesai (`stop`) → banner "Pindahkan ke Review?". Pengguna yang memutuskan; `Done` hanya oleh manusia.
- Ringkasan per proyek: agent anggota (+ aktivitas), kartu per kolom, biaya total sesi tertaut, kartu macet (In Progress tanpa aktivitas agent > N jam).
- Opsional: impor/sinkron dengan GitHub Issues/Projects.

### 7.1 Template alur & relasi kartu

- Dari kartu `bug` (atau tombol **"Buat dari tes gagal"**) pengguna memilih template **Siklus bug**; sistem membuat rantai `Fix → Deploy → Retest → Update dokumen` sebagai kartu terhubung (`blocks`) dengan peran masing-masing, lalu **meminta konfirmasi** sebelum menyimpan. Kartu bisa diedit/dihapus setelah dibuat (tidak semua bug butuh dokumen).
- Aturan buka-blokir pada bagian 1.2 dijalankan oleh TaskService, tercatat sebagai event `task.moved` dengan alasan ("dependensi Fix terpenuhi").
- Tampilan **Rantai**: satu bug menampilkan seluruh perjalanannya (iterasi 1, 2, 3 …) dan status tiap langkah; filter papan per peran/jenis/iterasi.
- Kartu `Deploy`/`Retest` menyimpan `env` dan `build`/commit; pengisian manual di v1 (integrasi CI/GitHub Actions menyusul).

### 7.2 Peran, antrean, dan banyak agent per tim

- Kartu dapat ditugaskan ke **anggota tertentu** atau ke **peran** (antrean). Anggota berperan itu melihat antrean di inbox-nya; "Ambil" = menugaskan ke dirinya (atau agent mengklaim lewat MCP pada tahap kendali).
- Satu sesi bisa memegang peran berbeda di waktu berbeda: peran yang berlaku mengikuti **kartu fokus**, bukan label agent.
- **Anggota manusia** tampil sama seperti agent di papan, tetapi tanpa sesi/monitoring. Cara statusnya terisi, berurutan menurut prioritas pembangunan: (1) **QA sebagai proxy** menggeser kartu atas nama mereka (v1); (2) **sinyal Git/CI** — MR/PR dibuka atau di-merge dan pipeline selesai menghasilkan *saran* perpindahan (7.5); (3) **akun login ringan** dengan view "Kartu saya" dan notifikasi (setelah v1).
- Kartu yang menunggu manusia ditandai `menunggu orang`, dan tidak dihitung sebagai "agent macet".
- Notifikasi (email/Slack/Telegram) untuk kartu peran/antrean: kanal belum ditentukan (lihat pertanyaan terbuka).

### 7.3 Jurnal & resume harian (Notion dan Google Docs)

Dokumen progres dan "resume tiap hari" adalah pekerjaan rutin; Mission Control memegang datanya agar tidak ditulis dari ingatan:
- **Digest harian otomatis (draf)** per proyek, dari data nyata: sesi hari itu (siapa, berapa lama, model, biaya), kartu yang berpindah kolom, bug baru/tertutup, hasil retest dan tes, kartu macet. Berlabel *dihasilkan*; tidak pernah dianggap final.
- Pengguna (atau agent berperan `docs`) menyunting draf menjadi **JournalEntry**. **Sumber kebenaran jurnal tetap di Mission Control**; Notion dan Google Docs adalah **target publikasi**.
- **Publisher** (antarmuka `JournalPublisher.publish(entry, target)`), dengan implementasi:
  | Publisher | Bentuk di target | Idempoten dengan |
  |---|---|---|
  | `notion` | satu halaman per hari per proyek dalam sebuah database (properti: Proyek, Tanggal, Status), atau tambahan blok di halaman proyek | `remoteId` halaman |
  | `gdocs` | satu dokumen per proyek (atau per bulan); tiap hari ditambahkan sebagai bagian berjudul tanggal | `remoteId` dokumen + penanda bagian tanggal |
  | `markdown` | berkas `.md` hasil ekspor (unduh atau tulis ke folder) | nama berkas bertanggal |
- **Satu arah** (Mission Control → target) pada v1; edit manual di Notion/Docs tidak ditarik kembali. Publikasi ulang entri yang sama **memperbarui** halaman/bagian yang ada, tidak menduplikasi.
- **Publish adalah aksi keluar**: selalu lewat *pratinjau → konfirmasi*, tidak pernah otomatis tanpa izin eksplisit per proyek. Isi yang dikirim tidak memuat transkrip mentah, secret, atau path pribadi; bukti tes hanya berupa tautan.
- Satu entri dapat dipublikasikan ke lebih dari satu target (mis. Notion untuk tim, Google Docs untuk laporan).
- Kartu `Update dokumen` di akhir siklus bug menunjuk ke dokumen yang harus diperbarui (tautan Notion/Docs/repo) dan dikerjakan agent berperan `docs` atau manusia.
- Pembagian tugas: **daemon** menerbitkan digest terstruktur; **agent berperan `docs`** (yang sudah punya konektor Notion/Google Docs di lingkungannya) menyunting dokumen naratif yang lebih bebas. Keduanya tidak saling menimpa: daemon hanya menulis bagian yang ia buat (ditandai).

### 7.4 Playwright: hasil tes → bug → retest

- **Sumber hasil tes** (keduanya didukung, ditambahkan berurutan):
  1. *Lokal*: Collector di mesin QA memantau jalur hasil yang dikonfigurasi (reporter JSON Playwright: `--reporter=json`; opsional JUnit) dan folder `test-results/` untuk lampiran.
  2. *CI*: setelah pipeline GitLab/GitHub selesai, daemon mengambil artefak hasil lewat API (webhook pipeline → unduh artefak `results.json`).
- **Impor** menghasilkan `TestRun` dengan `env` dan `build` (commit SHA, dari CI atau diisi manual untuk run lokal). Status `flaky` (lulus setelah retry di Playwright) disimpan sebagai `flaky`, **tidak otomatis menjadi saran bug**, tetapi dihitung dalam statistik kestabilan tes.
- **Bug dari kegagalan**: kegagalan baru → saran "Buat bug" dengan judul tes, berkas, pesan galat, `env`, `build`, serta tautan screenshot/video/**trace**; pengguna mengonfirmasi lalu memilih template **Siklus bug**. Fingerprint mencegah duplikat: kegagalan yang sama muncul lagi → bug yang ada mendapat bukti baru, bukan kartu baru.
- **Retest berbasis bukti**: kartu `Retest` menyimpan `testIds[]` yang harus lulus. Ketika `TestRun` baru pada `env` dan `build` yang sesuai memuat tes-tes itu, sistem *menyarankan* hasil `lulus` atau `gagal` dengan tautan ke run-nya; QA mengonfirmasi. Run di `env`/`build` yang berbeda dari Deploy tidak dianggap bukti (diberi peringatan).
- **Peringatan keamanan bukti**: trace dan video Playwright dapat memuat data jaringan, token, atau data pengguna. Secara bawaan hanya **tautan** yang disimpan, bukan salinan; lampiran tidak pernah dikirim ke Notion/Google Docs tanpa pratinjau.
- Ringkasan kesehatan tes per proyek: lulus/gagal/flaky per run, tren, tes paling sering gagal.

### 7.5 Git dan CI/CD (GitHub dan GitLab)

Integrasi memberi **sinyal**; sinyal menghasilkan *saran*, bukan perubahan status diam-diam.
| Peristiwa | Sinyal → saran |
|---|---|
| Branch/MR/PR menyebut kartu (`MC-123` pada judul/branch/deskripsi) | tautkan MR/PR ke kartu (`externalRefs`) |
| MR/PR dibuka oleh anggota (dipetakan lewat `externalIdentities`) | kartu `Fix` → *In Progress* (saran) |
| MR/PR di-merge | kartu `Fix` → *Review/Done* (saran; `Done` tetap manusia) |
| Pipeline job deploy sukses untuk `env` tertentu | kartu `Deploy` → saran *Done*, mengisi `env` dan `build` (commit SHA) otomatis |
| Pipeline gagal pada MR | tanda merah di kartu `Fix`, tautan ke log pipeline |
- Dukungan **GitHub (Actions)** dan **GitLab (CI/CD, termasuk self-hosted)** lewat satu antarmuka `GitProvider` (`webhook`, `getArtifact`, `getPipeline`, `getMergeRequest`); webhook diverifikasi dengan secret bersama. Nama job deploy per `env` dikonfigurasi per proyek.
- Karena kita tidak ingin bergantung pada akses masuk ke laptop QA, webhook diterima oleh Daemon (VPS) atau lewat Collector (polling API) bila Daemon hanya lokal.

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
- **Integrasi eksternal**: token Notion/Google/GitHub/GitLab disimpan terenkripsi (keychain OS/secret store), dengan cakupan izin sekecil mungkin (Notion: hanya database/halaman yang dibagikan; Google: akses ke dokumen yang dipilih; Git: baca MR/pipeline/artefak). Webhook wajib bertanda tangan. Setiap penerbitan ke luar (Notion/Docs) lewat pratinjau + konfirmasi, tercatat di audit, dan tidak memuat transkrip mentah/secret/path pribadi. Artefak tes (trace/video/screenshot) diperlakukan sensitif: hanya tautan, tidak disalin.

## 10. Roadmap

Urutan mengikuti prioritas: monitoring → proyek & assignment → kanban per proyek → 3D → kendali.

| Fase | Isi | Selesai bila |
|---|---|---|
| 0 | Repo, Daemon+Collector kerangka, SQLite, SSE, skema `MissionEvent`, fixture dari sampel | event dari fixture ter-stream ke UI |
| 1 | **Monitoring**: watcher + parser OMP (v3) dan Claude Code; daftar Agent & Sesi dengan aktivitas, model, token/biaya | sesi nyata dari kedua provider tampil & berubah status |
| 2 | **Proyek & assignment**: CRUD proyek, pemetaan `cwd`→proyek, Inbox proyek baru, assign agent↔proyek, penanda "bekerja di luar proyek" | agent bisa di-assign dan sesinya muncul di proyek yang benar |
| 3 | **Kanban per proyek** (v1 tanpa menjalankan agent): kartu, kolom, anggota (agent/orang) dan peran, antrean peran, segmen tautan kartu↔sesi + Fokus, aktivitas tanpa kartu | satu proyek dikelola penuh dari papan |
| 3b | **Siklus bug**: jenis kartu, relasi `blocks/retestOf`, template alur, aturan buka-blokir, tampilan Rantai, `env/build` | satu bug berjalan Bug→Fix→Deploy→Retest→Update dokumen dengan aturan otomatis yang tercatat |
| 3c | **Jurnal harian**: digest otomatis (draf), JournalEntry, Publisher `markdown`, lalu `notion` dan `gdocs` (satu arah, pratinjau → konfirmasi, idempoten) | digest hari ini akurat dan terbit ke Notion/Google Docs tanpa duplikasi |
| 3d | **Playwright**: impor `results.json` (lokal via Collector, lalu dari CI), `TestRun`, fingerprint, bug dari kegagalan, retest berbasis bukti | kegagalan tes menjadi bug dengan bukti; retest disarankan lulus/gagal dari run baru |
| 3e | **GitHub/GitLab**: webhook MR/PR/pipeline, tautan kartu↔MR, saran perpindahan, `Deploy` terisi `env/build` dari job CI | MR di-merge dan job deploy sukses memunculkan saran yang benar di kartu |
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

Sudah dijawab: bangun baru; lokal dan VPS; karakter campuran; OMP = Oh My Pi; konsep inti = monitoring → assign agent ke proyek → kanban per proyek; satu sesi memegang beberapa pekerjaan/peran; alur bug QA (1.2); **tim FE campuran agent dan manusia**; **Playwright** untuk tes automation; **GitLab dan GitHub CI/CD**; **jurnal di Notion dan Google Docs**.

1. **Repo & CI per proyek**: tiap proyek di GitHub atau GitLab (atau campuran)? GitLab self-hosted atau gitlab.com? Menentukan konfigurasi `GitProvider` dan jalur akses webhook.
2. **Hasil Playwright**: reporter apa yang sudah aktif (`json`, `html`, `junit`, `allure`)? Apakah `results.json` diunggah sebagai artefak pipeline, atau hanya ada di laptop QA? Bukti (trace/video) dipakai rutin?
3. **Lingkungan & deploy**: daftar `env` (dev/staging/prod?) dan nama job deploy tiap `env` di pipeline; apakah deploy ke staging otomatis saat merge atau dipicu manual?
4. **Struktur Notion**: satu database "Jurnal" (satu halaman per hari per proyek) atau halaman per proyek dengan blok per hari? **Google Docs**: satu dokumen per proyek atau per bulan? Sudah ada template yang harus diikuti?
5. **Anggota manusia**: siapa saja yang perlu akun login (v1.x)? Kanal notifikasi yang dipakai tim (email, Slack, Telegram, WhatsApp)?
6. **Penomoran kartu** yang dipakai di judul branch/MR (mis. `MC-123`, atau mengikuti tracker lain seperti Jira/Huly)? Menentukan pola penautan MR ↔ kartu.
7. **Daftar peran proyek**: cukup `qa, fe, be, docs, pm`, atau ada peran lain (`devops`, `ba`)?
8. **Identifikasi agent**: otomatis `provider + host` (bisa diganti nama/digabung) — setuju?
9. **Pembuatan proyek** dari `cwd` baru: masuk Inbox lalu dikonfirmasi, atau manual dulu?
10. **Sampel OMP tambahan** (izin `always-ask`, `task`/subagent, `edit`, `omp acp`) — hanya untuk fase kendali dan subagent (lihat `omp-session-format.md` bagian 6).
11. Aset karakter: model gratis berlisensi terbuka atau dibuat khusus?
