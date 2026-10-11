# Glossary Domain — Mission Control

> Tujuan: istilah yang dipakai stakeholder dan istilah di kode harus bisa
> dipetakan satu sama lain. Ketidakcocokan di sini adalah sumber bug yang
> tidak kelihatan sampai terlambat.

| Istilah (domain) | Di kode | Arti | Catatan |
|---|---|---|---|
| Agent | `Agent` | Identitas persisten, bukan satu sesi | Default: provider+host; tampil sebagai karakter 3D |
| Session | `Session` | Satu run provider untuk satu task | Punya `providerSessionId` |
| Kartu / Task | `Task` | Unit kerja milik satu proyek | Worktree hanya saat tahap kendali |
| Connection / Adapter | `AgentConnection` | Penghubung ke satu jenis provider | Menghasilkan `MissionEvent` |
| Event | `MissionEvent` | Kejadian ternormalisasi | Penggerak animasi, kanban, log |
| Activity | `AgentActivity` | Keadaan agent saat ini | idle, thinking, tool_use, responding, waiting_input, waiting_permission, error |
| Approval | `ApprovalBroker` | Izin manusia untuk aksi berisiko | Dari hook izin / `canUseTool` |
| Collector | `collector/` | Proses di sisi agent yang menjalankan adapter dan mengirim event ke daemon | Pada mode lokal satu proses dengan daemon |
| Host | `hostId` | Mesin tempat agent berjalan | Worktree dan eksekusi terjadi di host ini |
| Project | `Project` | Satu repo/folder kerja; punya papan kanban sendiri | Dikenali dari `cwd` sesi |
| Assignment | `Assignment` | Penugasan agent ke proyek (niat) | Berbeda dari proyek tempat sesi benar-benar berjalan (`cwd`) |
| Bekerja di luar proyek | status turunan | Sesi agent berjalan di `cwd` proyek lain dari assignment-nya | Ditampilkan, tidak dikoreksi otomatis |
| Aktivitas tanpa kartu | status turunan | Sesi di proyek yang belum tertaut ke kartu | Bisa dijadikan/ditautkan ke kartu |
| Room / Blueprint | `Room` | Ruangan 3D per proyek, didefinisikan JSON | Data-driven |
| Review | kolom kanban | Agent sudah selesai, menunggu diterima manusia | Selesai oleh agent belum berarti Done |
| Stale | status turunan | Sesi aktif tanpa kabar terlalu lama | Bukan dianggap selesai |

## Istilah yang gampang tertukar

| | Bukan | Bedanya |
|---|---|---|
| | | |
