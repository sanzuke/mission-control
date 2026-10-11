# ADR-0001: Node.js tanpa dependensi + satu index.html, ledger JSONL sebagai sumber kebenaran

- **Status:** Proposed
- **Tanggal:** 2026-10-11
- **Pengambil keputusan:** @sanzuke

## Context

Pemilik sudah menulis spesifikasi Mission Control (Google Doc referensi): backend Node tanpa dependensi, frontend satu `index.html` dengan Three.js r128 lokal, data JSON/JSONL tanpa database, polling 3 detik, dan prinsip "tidak ada data palsu". Target dijalankan di VPS/container kecil dan dibuka lewat tunnel. Kebutuhan tambahan: kanban, integrasi Claude Code, dan provider lain (OMP, dst.).

Versi awal rancangan ini mengusulkan stack React + react-three-fiber + SQLite + Claude Agent SDK; itu diganti karena bertentangan dengan batasan referensi.

## Options considered

1. **Zero-dependency Node + single-file HTML + ledger JSONL (sesuai referensi)** - mudah di-deploy dan diaudit, tanpa build/npm. Kekurangan: tanpa framework dan loader GLB; kode UI besar dalam satu file.
2. **React + R3F + SQLite + SDK** - ergonomi dan ekosistem lebih baik, tetapi menambah build step, npm, dan melanggar batasan referensi.

## Decision

Opsi 1. Ledger `data/tasks.jsonl` adalah sumber kebenaran status agent; semua provider (Claude Code lewat hooks dan headless `claude -p --output-format stream-json`, OMP, generic-cli) hanya meng-append event ke sana lewat adapter kecil. Kanban disimpan di `data/board.json` dan terhubung ke ledger lewat `task_id`. Karakter robot dibangun prosedural dari primitif Three.js.

## Consequences

**Yang jadi lebih mudah:**
- Deploy `node server.js`; tidak ada npm/build; mudah diverifikasi dengan `node --check` dan `curl`.
- Provider baru = satu adapter yang menulis event ledger.

**Yang jadi lebih sulit / harga yang kita bayar:**
- Tanpa SDK Claude, kita mem-parse stream-json sendiri; format bisa berubah antarversi CLI.
- Polling 3 detik, bukan push; animasi bisa tertinggal beberapa detik.
- JSONL tumbuh terus (perlu rotasi) dan tulis konkuren perlu `appendFile` per baris.
- Satu `index.html` besar sulit dirawat; karakter prosedural kurang detail dibanding GLB.

**Yang perlu dipantau:**
- Jika `index.html` melewati sekitar 3000 baris atau perlu GLB/animasi skeletal, pecah jadi modul atau izinkan loader yang di-vendor.
- Jika perlu kendali dua arah real-time (approval), tinjau SSE/WebSocket.
