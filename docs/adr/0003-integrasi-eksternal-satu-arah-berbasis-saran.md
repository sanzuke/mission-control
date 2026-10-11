# ADR-0003: Integrasi eksternal bersifat satu arah dan berbasis saran; Mission Control tetap sumber kebenaran

- **Status:** Proposed
- **Tanggal:** 2026-10-11
- **Pengambil keputusan:** @sanzuke

## Context

Alur kerja pemilik melibatkan layanan di luar Mission Control: tes Playwright, GitHub/GitLab (MR/PR, pipeline CI/CD), serta jurnal di Notion dan Google Docs. Tim FE terdiri dari agent AI dan manusia; manusia bekerja di Git dan tidak selalu membuka dashboard. Data di layanan luar bisa berubah di luar kendali kita, dan sebagian (trace/video tes, transkrip) sensitif.

## Options considered

1. **Sinkron dua arah penuh dan otomatis** (status kartu mengikuti MR/pipeline, dokumen ditarik balik dari Notion/Docs) - minim kerja manual; rawan konflik, status salah diam-diam, dan sulit diaudit.
2. **Satu arah: Git/CI/tes memberi sinyal yang menghasilkan saran; jurnal diterbitkan ke Notion/Docs oleh Publisher; Mission Control tetap sumber kebenaran** - mudah diaudit dan dibatalkan; masih ada klik konfirmasi.
3. **Tanpa integrasi** (semua manual) - sederhana, tetapi manusia di tim FE dan hasil tes tidak terwakili.

## Decision

Opsi 2. Sinyal dari Git/CI/Playwright hanya membuat *saran* (perpindahan kartu, bug baru, hasil retest) yang dikonfirmasi manusia; `Done` tetap manusia. Jurnal disimpan di Mission Control dan diterbitkan lewat `JournalPublisher` (Notion, Google Docs, Markdown) secara idempoten dan satu arah, dengan pratinjau dan konfirmasi. Artefak tes dan transkrip tidak disalin keluar; hanya tautan. Kredensial integrasi hanya berupa referensi ke secret store. GitHub didukung lebih dulu (GitLab menyusul lewat antarmuka yang sama); Telegram adalah satu-satunya kanal notifikasi dan hanya untuk notifikasi, komentar, serta aksi berisiko rendah (ambil/mulai); keputusan penting tetap lewat web. Retest hanya sah bila `fixCommit` adalah leluhur SHA yang dideploy.

## Consequences

**Yang jadi lebih mudah:**
- Setiap perubahan status dapat dijelaskan ("karena MR !42 di-merge") dan dibatalkan.
- Menambah integrasi baru (Jira, Slack) = satu adapter yang menghasilkan sinyal atau Publisher.

**Yang jadi lebih sulit / harga yang kita bayar:**
- Masih ada klik konfirmasi; mungkin terasa berulang untuk alur yang sudah pasti.
- Perubahan manual di Notion/Docs tidak kembali ke Mission Control; dua tempat bisa berbeda.
- Pemetaan identitas (penulis MR -> anggota) dan penautan MR -> kartu perlu dirawat.

**Yang perlu dipantau:**
- Bila sebagian besar saran selalu dikonfirmasi tanpa diubah, pertimbangkan mode otomatis per aturan (opt-in per proyek) untuk jenis saran tertentu.
- Bila tim sering mengedit jurnal langsung di Notion, pertimbangkan sinkron balik terbatas.
