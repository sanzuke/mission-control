# ADR-0002: Tautan task-sesi berupa segmen berjangka waktu dengan Fokus sesi

- **Status:** Proposed
- **Tanggal:** 2026-10-11
- **Pengambil keputusan:** @sanzuke

## Context

Pemilik (QA manual/automation) menjalankan satu sesi agent yang menangani beberapa pekerjaan berurutan: menulis dokumen progres harian, memperbaiki FE, dan testing. Sebuah bug melewati beberapa kartu (Bug, Fix, Deploy, Retest, Update dokumen) yang dikerjakan peran dan sesi berbeda. Model "satu sesi untuk satu task" tidak cocok, dan biaya/aktivitas per task tidak bisa dihitung bila sesi dibagi banyak task.

## Options considered

1. **Satu sesi = satu task** - sederhana; salah untuk cara kerja nyata, dan memaksa membuka sesi baru tiap pergantian pekerjaan.
2. **Segmen berjangka waktu `(session, task, dari, sampai)` dengan Fokus sesi yang diganti manual/MCP, sistem hanya menyarankan** - akurat dan bisa diaudit; butuh disiplin mengganti fokus.
3. **Deteksi otomatis task dari isi percakapan/berkas** - tanpa usaha pengguna; tebakan bisa salah dan merusak data biaya/progres.

## Decision

Opsi 2. Tautan adalah segmen berbatas event; sesi punya `focusTaskId`. Pergantian fokus menutup segmen lama dan membuka yang baru. Sistem boleh menyarankan pergantian (perubahan `cwd`/berkas, `intent`) tetapi tidak mengubah tautan tanpa konfirmasi. Aktivitas sebelum ada fokus tercatat sebagai "tanpa kartu" dan dapat ditautkan belakangan (mengubah segmen, tercatat di audit).

## Consequences

**Yang jadi lebih mudah:**
- Biaya, durasi, dan aktivitas per kartu akurat walau sesi dipakai bergantian.
- Satu kartu bisa lintas sesi dan lintas agent; riwayat lengkap di tab Sesi.

**Yang jadi lebih sulit / harga yang kita bayar:**
- Pengguna perlu mengganti fokus; segmen yang terlewat harus dikoreksi belakangan.
- Model data dan UI lebih rumit dari tautan satu-ke-satu; koreksi segmen harus menjaga konsistensi total biaya.

**Yang perlu dipantau:**
- Bila sebagian besar aktivitas tetap "tanpa kartu", pertimbangkan saran fokus yang lebih agresif (tetap dengan konfirmasi) atau tool MCP agar agent melaporkan fokusnya sendiri.
