# Glossary Domain — Mission Control

> Tujuan: istilah yang dipakai stakeholder dan istilah di kode harus bisa
> dipetakan satu sama lain. Ketidakcocokan di sini adalah sumber bug yang
> tidak kelihatan sampai terlambat.

| Istilah (domain) | Di kode | Arti | Catatan |
|---|---|---|---|
| Agent | `Agent` | Identitas persisten, bukan satu sesi | Default: provider+host; tampil sebagai karakter 3D |
| Session | `Session` | Satu run provider; boleh mengerjakan beberapa task berurutan | Punya `providerSessionId` dan `focusTaskId` |
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
| Fokus sesi | `Session.focusTaskId` | Task yang sedang dikerjakan sesi saat ini | Diganti manual/MCP; sistem hanya menyarankan |
| Segmen | `TaskLink` | Rentang event sesi untuk satu task | Dasar penghitungan biaya per task |
| Peran | `roles[]` | qa, fe, be, docs, pm … | Melekat pada assignment/kartu, bukan agent |
| Member | `Member` | Agent AI atau manusia yang bisa jadi assignee | Manusia tidak punya sesi/monitoring |
| Antrean peran | `assigneeRole` | Kartu untuk satu peran, belum diambil anggota tertentu | Diambil anggota berperan itu |
| Template alur | `WorkflowTemplate` | Rantai kartu terhubung dengan peran dan aturan buka-blokir | Bawaan: "Siklus bug" |
| Rantai / Iterasi | `TaskRelation`, `iteration` | Perjalanan satu bug dari temuan sampai lulus retest | Retest gagal → iterasi + 1 |
| Retest | kartu `kind=retest` | Pengujian ulang setelah deploy | Harus pada env+build yang sama dengan Deploy |
| Jurnal | `JournalEntry` | Catatan progres harian per proyek | Digest otomatis hanyalah draf; sumber kebenaran di Mission Control |
| Publisher | `JournalPublisher` | Penerbit jurnal ke target luar (Notion, Google Docs, Markdown) | Satu arah, pratinjau → konfirmasi, idempoten |
| TestRun / TestResult | `TestRun` | Satu eksekusi suite tes (mis. Playwright) beserta hasil per tes | Membawa `env` dan `build` |
| Flaky | `status=flaky` | Tes yang lulus setelah retry | Tidak otomatis jadi saran bug |
| Fingerprint bug | `fingerprint` | Hash tes + tanda tangan galat untuk mencegah duplikat | Kegagalan sama → bukti baru pada bug lama |
| Integration | `Integration` | Sambungan ke layanan luar (Notion, Google, GitHub, GitLab) | Kredensial hanya referensi ke secret store |
| Sinyal / Saran | status turunan | Peristiwa Git/CI/tes yang diubah menjadi usulan perpindahan kartu | Tidak mengubah status tanpa konfirmasi |
| Room / Blueprint | `Room` | Ruangan 3D per proyek, didefinisikan JSON | Data-driven |
| Review | kolom kanban | Agent sudah selesai, menunggu diterima manusia | Selesai oleh agent belum berarti Done |
| Stale | status turunan | Sesi aktif tanpa kabar terlalu lama | Bukan dianggap selesai |

## Istilah yang gampang tertukar

| | Bukan | Bedanya |
|---|---|---|
| | | |
