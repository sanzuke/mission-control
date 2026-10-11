# ADR-0001: Daemon TypeScript lokal + UI web (react-three-fiber) dengan adapter provider

- **Status:** Proposed
- **Tanggal:** 2026-10-11
- **Pengambil keputusan:** @sanzuke

## Context

Mission Control harus memvisualkan banyak agent sebagai karakter 3D, punya kanban, terintegrasi paling dalam dengan Claude Code, dan tetap bisa menyambung ke OMP serta provider lain. Repo masih kosong; belum ada batasan sistem lama.

## Options considered

1. **Daemon Node/TS + UI web (three.js/R3F)** — satu bahasa, Claude Agent SDK tersedia di TS, 3D web paling matang. Kekurangan: tak ada akses native ke GPU/OS tanpa pembungkus.
2. **Aplikasi desktop native (Tauri/Electron sejak awal)** — pengalaman lebih "app", tapi menambah beban build/rilis sebelum produk terbukti.
3. **Game engine (Unity/Godot)** — visual terbaik, tetapi integrasi proses/SDK/kanban jauh lebih berat dan sulit dikontribusikan.

## Decision

Opsi 1: daemon lokal Node/TS (event bus, SQLite, hook receiver, MCP server) dan UI web React + react-three-fiber. Semua provider masuk lewat interface `AgentProvider` yang menormalkan event ke `MissionEvent`. Pembungkus desktop (Tauri) bisa ditambah belakangan tanpa mengubah inti.

## Consequences

**Yang jadi lebih mudah:**
- Menambah provider = satu adapter; UI tak berubah.
- Integrasi Claude Code (SDK, hooks, MCP) tinggal di satu bahasa.

**Yang jadi lebih sulit / harga yang kita bayar:**
- Performa 3D dibatasi browser; perlu mode lite.
- Skema `MissionEvent` harus dijaga stabil; perubahan memengaruhi semua adapter.
- Fitur provider tak seragam, sehingga UI harus mengikuti `capabilities`.

**Yang perlu dipantau:**
- Bila 3D di browser tak mencapai 60 fps dengan >6 agent, tinjau ulang (OffscreenCanvas/WebGPU atau desktop native).
- Setelah arti "OMP" terkonfirmasi: bila tak bisa lewat CLI/ACP, tinjau interface adapter.
