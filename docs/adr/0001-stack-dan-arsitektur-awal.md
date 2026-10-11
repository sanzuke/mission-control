# ADR-0001: Daemon Node/TS + UI React/R3F, event ternormalisasi lewat antarmuka AgentConnection

- **Status:** Proposed
- **Tanggal:** 2026-10-11
- **Pengambil keputusan:** @sanzuke

## Context

Mission Control harus memvisualkan banyak agent sebagai karakter 3D, punya kanban, terintegrasi paling dalam dengan Claude Code, dan tetap bisa menyambung ke OMP serta provider lain. Dua referensi diperiksa: (1) Google Doc yang ternyata prompt untuk muse.ai (zero-dependency, polling, satu index.html) - hanya ide fitur, bukan batasan; (2) proyek open source CrewHub (FastAPI + React/Three.js + SQLite + SSE, lisensi AGPL-3.0) yang membuktikan pendekatan transkrip JSONL + SSE + dunia 3D bekerja.

Riwayat: draf awal memilih Node/TS + R3F; draf kedua sempat pindah ke zero-dependency mengikuti dokumen muse.ai; setelah diklarifikasi bahwa dokumen itu bukan spesifikasi, keputusan kembali ke opsi 1 dengan pelajaran dari CrewHub.

Cakupan produk (klarifikasi pemilik): monitoring agent adalah inti; agent di-assign ke proyek; setiap proyek punya task management sendiri. Menjalankan agent dari dashboard ditunda ke tahap kendali, sehingga v1 hanya mengamati (Collector tidak perlu men-spawn proses).

## Options considered

1. **Node/TS daemon + React/R3F + SQLite + SSE** - satu bahasa, Claude Agent SDK tersedia di TS, ekosistem 3D web terbesar. Kekurangan: butuh build step dan npm.
2. **Python FastAPI + React/R3F (seperti CrewHub)** - terbukti, `psutil`/proses mudah. Kekurangan: dua bahasa; SDK Claude TS tak dipakai langsung.
3. **Zero-dependency Node + satu index.html (gaya muse.ai)** - paling ringan di-deploy; sulit dirawat untuk 3D, kanban, dan banyak adapter.
4. **Fork CrewHub** - tercepat; terikat AGPL-3.0 dan arah proyek mereka.

## Decision

Opsi 1. Setiap provider mengimplementasikan `AgentConnection` dan mengeluarkan `MissionEvent`; UI hanya bergantung pada `capabilities`. Claude Code dipantau lewat transkrip JSONL + hooks dan dikendalikan lewat SDK/proses `claude`. Kode ditulis dari nol (tidak menyalin CrewHub) untuk menghindari kewajiban AGPL. Karena penggunaan lokal dan VPS sekaligus, adapter berjalan di proses **Collector** di sisi agent dan mengirim event ke Daemon pusat (satu proses pada mode lokal). Untuk OMP (Oh My Pi), kendali memakai `omp acp` (ACP via stdio) dengan watcher berkas sesi untuk observasi.

## Consequences

**Yang jadi lebih mudah:**
- Menambah provider = satu adapter; UI tidak berubah.
- Satu bahasa untuk daemon, UI, dan tipe event.

**Yang jadi lebih sulit / harga yang kita bayar:**
- Dua proses (Collector dan Daemon) berarti protokol, autentikasi per-collector, dan penanganan koneksi putus harus dirancang; sinkronisasi repo antar-host bukan urusan kita.
- Format transkrip/stream tiap CLI berubah antarversi; perlu golden-file test per provider.
- Fitur provider tidak seragam sehingga UI harus mengikuti `capabilities`.
- Ada build step dan dependensi npm; mode Docker hanya bisa monitor.
- Performa 3D di browser dibatasi; perlu mode Zen/2D.

**Yang perlu dipantau:**
- Jawaban atas "bangun baru vs fork CrewHub" dan verifikasi mode RPC OMP (Oh My Pi); bila OMP tak bisa lewat CLI/ACP, tinjau antarmuka adapter.
- Bila 3D tak mencapai 60 fps dengan >6 agent, tinjau OffscreenCanvas/WebGPU.
