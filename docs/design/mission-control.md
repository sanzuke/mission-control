# Rancangan Mission Control

> Status: draft v0.2 · 2026-10-11 · dokumen living (edit di tempat, riwayat lewat `git log -p`)
> v0.2: diselaraskan dengan spesifikasi referensi pemilik (Google Doc "Mission Control", 9 menu, ledger JSONL, tanpa dependensi). Kanban, adapter multi-provider, dan integrasi Claude Code adalah **tambahan** di atas spesifikasi itu.

## 1. Tujuan

Pusat kendali web untuk memantau dan mengarahkan tim AI agent secara real-time. Agent tampil sebagai **robot 3D** di Virtual Office, pekerjaannya tercatat di **ledger**, dan perencanaan kerja dikelola di **kanban**.

1. **Claude Code** = integrasi paling dalam.
2. **OMP** dan provider lain lewat adapter seragam.
3. **Kejujuran data**: tidak ada angka/status simulasi. Kalau sumber data tidak tersedia, tampilkan "tidak tersedia", bukan nol atau placeholder.

Bukan tujuan v1: editor kode di UI, multi-user publik, database.

## 2. Prinsip teknis (dari referensi)

- Backend: Node.js **tanpa dependensi eksternal** (`http`, `fs`, `child_process`), satu `server.js`, bind `127.0.0.1:3001`.
- Frontend: satu `index.html` (HTML+CSS+JS inline), tanpa framework. Three.js r128 di-vendor di `vendor/three.min.js` (tanpa CDN).
- Data: file JSON/JSONL di `data/`, tanpa database.
- Update: frontend polling `GET /api/state` tiap 3 detik.
- Tema: dark netral ala Linear/Vercel, kartu rounded 16px, badge pill; tanpa neon.

Konsekuensi: karena Three.js r128 tidak membawa loader GLB di core dan kita menghindari dependensi, **karakter dibangun prosedural dari primitif** (robot dari box/sphere/cylinder + aksesori per peran). Impor GLB/VRM ditunda (fase lanjut).

## 3. Arsitektur

```
Browser: index.html (Three.js scene · 10 menu)  ──poll 3s──►  GET /api/state
                                                 ──aksi────►  POST /api/queue | /api/dispatch | /api/board
                                    ▲
                           server.js (zero-dep)
   ┌───────────────┬───────────────┼────────────────┬──────────────────┐
 ledger reader   board store    system probes    schedule scanner    adapters/
 tasks.jsonl     board.json     /proc, df, uptime cron.d/*.md        claude-code · omp · generic-cli
   ▲
   │ append event (started|progress|done|failed)
 Claude Code hooks · wrapper adapter · tugas manual dari agent
```

**Ledger adalah sumber kebenaran status agent.** Semua adapter pada akhirnya hanya menambah baris ke `data/tasks.jsonl`; server hanya membaca. Ini membuat provider baru murah dan menjaga prinsip "data nyata".

### Struktur folder
```
server.js            index.html          watchdog.sh        PUBLIC_URL.txt
vendor/three.min.js
adapters/            claude-code.js  omp.js  generic-cli.js
hooks/               mc-hook.sh          # dipasang ke .claude/settings.json
data/                tasks.jsonl  queue.jsonl  dispatch.json  board.json  agents.json
docs/
```

## 4. Data (ledger)

Sesuai referensi:
- `data/tasks.jsonl`: `{ts, task_id, agent, subagent, title, status: started|progress|done|failed, detail}`. `task_id` stabil; `agent` wajib id roster.
- `data/queue.jsonl`: `{ts, prompt, agent, status: queued|done}`.
- `data/dispatch.json`: `{"paused": false}`.

Tambahan (kompatibel mundur, field opsional):
- `tasks.jsonl` boleh membawa `provider` (`claude-code|omp|...`), `session_id`, `tokens`, `cost`, `card_id` (menautkan ke kartu kanban).
- `data/board.json`: `{columns:[...], cards:[{id, title, description, column, agent, labels, priority, deps, task_id, branch, createdAt}]}` ditulis atomik (tulis ke `.tmp` lalu `rename`).
- `data/agents.json`: roster (id, nama, peran, provider, model, ciri visual) agar roster dapat diubah tanpa menyentuh kode.

Event tak dikenal/baris rusak di-skip tanpa crash.

## 5. API

- `GET /api/state` → `agents, queue, tasks, board, system, vps, schedules, dispatch_paused, log` (+ `tokens` bila tersedia).
- `POST /api/queue` `{prompt, agent}` · `POST /api/dispatch` `{paused}` (sesuai referensi).
- `POST /api/board/card` · `PATCH /api/board/card/:id` (pindah kolom, assign, edit) · `DELETE`.
- `POST /api/agent/:id/pause|resume` (klik karakter di 3D).
- Semua `POST` hanya dari origin yang sama + token sederhana bila diakses lewat tunnel publik.

Status agent diturunkan, bukan disimpan: `working` jika event terakhirnya `started|progress` dan belum `done|failed`; `paused` jika dispatch/agent dijeda; selain itu `idle`. Task `started` tanpa kabar > N menit ditandai `stale` (bukan diam-diam dianggap selesai).

## 6. Integrasi Claude Code (utama)

Semua jalur menulis ke ledger, jadi tidak butuh SDK/npm:

1. **Hooks (amati):** `hooks/mc-hook.sh` dipasang ke `.claude/settings.json` untuk `SessionStart, PreToolUse, PostToolUse, Notification, SubagentStop, Stop`. Skrip membaca JSON stdin dan meng-append event ke `tasks.jsonl` (`started` saat sesi mulai, `progress` per tool, `done` saat `Stop`). Session `claude` manual di terminal pun ikut tampil. `subagent` = id subagent Claude (sesuai field `subagent` di ledger).
2. **Headless runner (kendali):** adapter `claude-code.js` men-spawn `claude -p --output-format stream-json --resume <id>` lewat `child_process`, `cwd` = git worktree kartu, parsing JSONL stream → ledger. Dipicu saat item `queue.jsonl` dispatch atau kartu kanban di-"Start". Menghormati `dispatch.json` (pause).
3. **MCP kanban (opsional, fase lanjut):** server MCP stdio kecil (JSON-RPC manual, tanpa dependensi) dengan tool `list_cards`, `claim_card`, `update_card`, `add_subtask`, `comment_card` agar Claude bisa mengambil dan memperbarui kartu sendiri.
4. Token/biaya: dari field `usage`/`total_cost_usd` di stream-json; menu Token Status membaca ini bila ada, selain itu tampil "tidak tersedia".

## 7. Lapisan provider

Kontrak adapter (modul CommonJS kecil):
```js
module.exports = {
  id: 'claude-code',
  capabilities: { observe: true, run: true, interrupt: true, approvals: false, usage: true },
  run({ card, prompt, cwd, agent }) { /* spawn; return { pid, stop() } */ },
};
// Kewajiban: semua output dinormalisasi menjadi baris ledger via appendEvent().
```
| Adapter | Cara | Catatan |
|---|---|---|
| `claude-code` | hooks + headless runner | fitur penuh |
| `omp` | **asumsi**: proses CLI dengan keluaran JSON-lines/RPC | perlu konfirmasi apa itu OMP |
| `generic-cli` | spawn perintah + pola parse yang dikonfigurasi di `agents.json` | fallback agent apa pun |
| `api` | loop HTTP ke endpoint kompatibel OpenAI | ditunda, butuh loop tool-use sendiri |

UI membaca `capabilities`: tombol yang tidak didukung provider disembunyikan.

## 8. Menu & tampilan

Sidebar kiri + topbar, 10 menu (9 dari referensi + Kanban):
1. **Virtual Office** (default): ruangan 3D (meja, 2 monitor — kode scrolling & chart animasi —, kursi, jendela kota malam, tanaman, lampu status). Agent bekerja → **duduk di meja** dengan animasi sesuai peran (mengetik / menggambar); idle → **berdiri di barisan belakang**. Transisi posisi smooth, kedip, idle sway, *cheer hop* saat selesai. Kamera drag-rotate + scroll-zoom. **Klik karakter = pause/resume.** Di HP: canvas fullscreen tanpa sisa hitam dan tidak bisa di-scroll (`100dvh`, `touch-action:none`, `overscroll-behavior:none`).
2. **Agent Overview**: kartu per agent (working/idle/paused + task aktif).
3. **Kanban** *(baru)*: papan Backlog → Ready → In Progress → Review → Done (+ Blocked); drag kartu, assign ke agent, tombol Start.
4. **Task Queue**: antrean quick task + badge + tombol Research/Coding/Vision/Design.
5. **System Metrics**: CPU load, memory, uptime dari `/proc`, 3 detik.
6. **VPS Status**: 7 kartu (Disk ≥90% merah, CPU, Memory, Uptime, Tunnel ONLINE/OFFLINE, OS, Watchdog terakhir).
7. **Tugas Terjadwal**: dari frontmatter YAML `.md` (parser sederhana tanpa library), "Jadwal Saya" di atas, "Jadwal Sistem" collapsible, label Indonesia, waktu WIB.
8. **Token Status**: pemakaian vs batas paket bila terbaca; selain itu "tidak tersedia" yang mudah disambung.
9. **Agent Fleet**: daftar agent + kemampuan.
10. **Real-Time Log**: tail `tasks.jsonl`/log.

Memory VPS: karena di container, pakai jumlah RSS semua proses (`/proc/<pid>/statm` kolom resident × page size), label **"MEMORY (CONTAINER)"** + subteks `container kita • host: X / Y GB (di luar kendali)`. Bukan PSS (`smaps_rollup` tidak selalu terbaca).

### Mapping event → animasi robot
| Kondisi (dari ledger) | Visual |
|---|---|
| tak ada task aktif | berdiri di barisan belakang, idle sway |
| `started`/`progress` | pindah ke meja, animasi sesuai peran |
| `detail` menyebut tool Edit/Write | mengetik cepat; Read/Grep → menoleh ke monitor; Bash → lampu meja berkedip |
| `done` | cheer hop, lalu kembali ke barisan |
| `failed` / `stale` | kepala menunduk, lampu status merah |
| paused | membeku, ikon jeda |

Peran → properti visual (dari roster): `aris` baret ungu + stylus + tablet; `udin` headphone + tema merah REC; `vision-agent-03` mata kamera; `research-agent-01` kaca pembesar/antena; `coding-agent-02` keyboard; `muse` robot default.

## 9. Kanban ↔ ledger

Kartu dan event dihubungkan lewat `task_id`/`card_id`:
- Start kartu → buat worktree+branch `mc/<id>-<slug>`, append `started`, kartu → In Progress.
- `progress` → kartu tetap, tampil detail terakhir.
- `done` → kartu ke **Review** (bukan Done): selesai oleh agent ≠ diterima manusia. Tombol *Terima → Done* / *Minta revisi* (kirim komentar kembali sebagai prompt baru).
- `failed` atau `stale` → **Blocked** dengan alasan.
- Dependensi: kartu `Ready` hanya bisa Start bila dependensinya `Done`. WIP limit per agent (default 1).

## 10. Akses publik & operasional

- **bore** meng-expose :3001, URL di `PUBLIC_URL.txt`.
- `watchdog.sh` (cron 5 menit): cek tunnel; mati/URL berubah → restart, update `PUBLIC_URL.txt`, catat ke `watchdog-history.jsonl`, laporkan URL baru.
- Aturan: baca `PUBLIC_URL.txt` dan live-check HTTP 200 sebelum membagikan link.
- Karena bisa publik: wajib token untuk `POST` dan opsi basic-auth pada seluruh UI; secret provider tidak pernah masuk ledger atau `/api/state`; kendali hanya mengeksekusi perintah yang ada di whitelist adapter (tidak ada shell bebas dari UI).

## 11. Roadmap

| Fase | Isi | Selesai bila |
|---|---|---|
| 0 | `server.js` + `/api/state` + data nyata (system, vps) + folder `data/` | `curl /api/state` menampilkan field nyata |
| 1 | `index.html`: layout, tema, menu Agent Overview/Queue/Metrics/VPS/Log/Fleet | semua menu tampil dengan data nyata |
| 2 | Virtual Office 3D: ruangan, 6 robot, state dari ledger, klik pause, mobile fullscreen | animasi cocok dengan event ledger |
| 3 | Hooks Claude Code → ledger | sesi `claude` manual muncul sebagai robot |
| 4 | Kanban + worktree + Start via headless runner | satu kartu end-to-end sampai Review |
| 5 | Jadwal (cron .md), Token Status, tunnel + watchdog | semua 10 menu selesai |
| 6 | Adapter OMP + generic-cli; MCP kanban; polish | dua provider jalan bersamaan |

Aturan kerja tiap fase (dari referensi): `node --check`, restart server, `curl /api/state`, pastikan field baru ada.

## 12. Pertanyaan terbuka

1. **OMP** = apa persisnya, dan bagaimana cara diintegrasikan (CLI/RPC)?
2. **Roster**: referensi minta konfirmasi nama/peran/ciri visual sebelum membangun. Pakai 6 agent di atas apa adanya, dan apa `coding-agent-02` itu Claude Code?
3. **Lokasi cron** selain `~/workspace/cron.d/` dan `~/workspace/goals/*/crons/`?
4. **Token Status**: perintah status langganan mana yang tersedia di server (mis. output `/usage`/`ccusage`)?
5. Apakah dashboard cukup *memantau + mengantrekan* (v1), atau harus ikut *menjalankan* agent (fase 4)?
