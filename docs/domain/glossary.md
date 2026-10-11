# Glossary Domain — Mission Control

> Tujuan: istilah yang dipakai stakeholder dan istilah di kode harus bisa
> dipetakan satu sama lain. Ketidakcocokan di sini adalah sumber bug yang
> tidak kelihatan sampai terlambat.

| Istilah (domain) | Di kode | Arti | Catatan |
|---|---|---|---|
| Agent | `Agent` | Identitas persisten AI coding (provider, model, persona, avatar) | Tampil sebagai karakter 3D |
| Session | `session_id` | Satu run provider untuk satu task | Opsional di ledger |
| Kartu | `board.json cards[]` | Unit kerja di kanban | 1 task = 1 git worktree |
| Provider | `adapters/*.js` | Backend agent (Claude Code, OMP, Codex, ...) | Masuk lewat adapter |
| Ledger | `data/tasks.jsonl` | Log append-only event kerja agent; sumber kebenaran status | status: started, progress, done, failed |
| Roster | `data/agents.json` | Daftar id agent yang sah | Field `agent` di ledger wajib id roster |
| Quick task | `data/queue.jsonl` | Antrean tugas cepat | Beda dengan kartu kanban (kerja terencana) |
| Dispatch pause | `data/dispatch.json` | Saklar jeda pengiriman tugas | Tombol PAUSE/RESUME |
| Review | kolom kanban | Agent sudah `done`, menunggu diterima manusia | `done` di ledger belum berarti Done di kanban |
| Stale | status turunan | Task `started` tanpa kabar terlalu lama | Bukan dianggap selesai |

## Istilah yang gampang tertukar

| | Bukan | Bedanya |
|---|---|---|
| | | |
