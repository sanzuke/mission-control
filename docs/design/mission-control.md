# Rancangan Mission Control

> Status: draft v0.1 · 2026-10-11 · dokumen living (edit di tempat, riwayat lewat `git log -p`)

## 1. Tujuan

Satu layar untuk memantau dan mengarahkan banyak AI coding agent. Setiap agent tampil sebagai **karakter 3D** yang bekerja di sebuah ruang kontrol, dan setiap pekerjaannya terhubung ke **kanban** sebagai sumber kebenaran task.

Prioritas:
1. **Claude Code** adalah integrasi kelas satu (paling dalam, paling lengkap).
2. **OMP** dan provider lain (Codex, Gemini CLI, API kompatibel OpenAI, dst.) lewat adapter yang seragam.
3. Visual 3D harus *informatif*, bukan sekadar hiasan: animasi karakter = status nyata agent.

Bukan tujuan (v1): menggantikan IDE, hosting multi-user publik, editor kode di dalam UI.

## 2. Arsitektur

```
┌─────────────── Web UI (React + react-three-fiber) ───────────────┐
│  Scene 3D (karakter, ruangan)  │  Kanban  │  Inspector  │  Log   │
└───────────────▲──────────────────────────────┬───────────────────┘
                │ WebSocket (event stream)     │ REST/tRPC (command)
┌───────────────┴──────────────────────────────▼───────────────────┐
│                 Mission Control Daemon (Node + TS)               │
│  Event Bus ─ Task Engine ─ Session Manager ─ Approval Broker     │
│  SQLite (tasks, sessions, events)   Git worktree manager         │
│  MCP server (kanban tools untuk agent)   Hook receiver (HTTP)    │
└──────┬───────────────┬────────────────┬──────────────────┬───────┘
       │               │                │                  │
  Claude Code adapter  OMP adapter   Codex/Gemini     OpenAI-compat
  (SDK + hooks + MCP)  (RPC/CLI)     (CLI/ACP)        (API loop)
```

Prinsip: **UI tidak pernah bicara langsung ke agent**. Semua lewat daemon yang menormalkan event ke satu skema `MissionEvent`. Itu yang membuat provider baru cukup menulis satu adapter.

### Stack usulan
| Lapisan | Pilihan | Alasan singkat |
|---|---|---|
| Daemon | Node + TypeScript, Fastify, `ws` | SDK Claude Agent berbasis TS; satu bahasa dengan UI |
| Storage | SQLite (better-sqlite3) + event log append-only | lokal, tanpa server, mudah di-backup |
| UI | React, Vite, Zustand | ringan |
| 3D | three.js via react-three-fiber + drei | ekosistem terbesar, GLB/VRM didukung |
| Karakter | VRM/GLB + animasi Mixamo / Blender | gratis/murah, bisa diganti user |
| Packaging | `npx mission-control` / Docker; Tauri opsional | mulai sebagai web lokal |

Lihat ADR-0001 untuk keputusan stack.

## 3. Model data

- **Project**: repo + konfigurasi.
- **Task** (kartu kanban): `id, title, description, status, priority, labels, assignee(agentId), parentId, worktreePath, branch, prUrl, acceptanceCriteria, createdAt…`
- **Agent**: identitas persisten (nama, provider, model, persona, avatar GLB, aturan izin).
- **Session**: satu run agent untuk satu task (`sessionId` provider, status, biaya/token, transcript ref).
- **MissionEvent** (append-only):
  `{ ts, sessionId, agentId, taskId, type, payload }`
  `type` ∈ `session.started | thinking | message | tool.start | tool.end | approval.requested | approval.resolved | subagent.started | subagent.ended | error | usage | session.ended`

## 4. Integrasi Claude Code (utama)

Tiga jalur, dipakai bersamaan:

1. **Mengendalikan session — Claude Agent SDK** (`@anthropic-ai/claude-agent-sdk`)
   Daemon menjalankan `query()` per task: set `cwd` = git worktree task, `resume` untuk lanjut session, `canUseTool` untuk approval, `abort` untuk interrupt. Stream pesan SDK → `MissionEvent`.
2. **Mengamati session yang dimulai manual — Hooks**
   Installer menulis hook ke `.claude/settings.json` (atau plugin) yang `POST` ke `http://127.0.0.1:<port>/hooks/claude`: `SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Notification, SubagentStop, Stop, SessionEnd`. Dengan ini terminal `claude` biasa pun muncul sebagai karakter di scene. `PreToolUse` bisa mengembalikan keputusan allow/deny/ask → dipakai Approval Broker.
3. **Agent ikut mengelola kanban — MCP server bawaan daemon**
   Tool: `list_tasks`, `claim_task`, `update_task`, `add_subtask`, `comment_task`, `request_review`, `link_pr`. Claude bisa “ambil kartu berikutnya” dan melaporkan progres tanpa glue-code.
   Opsional: baca `~/.claude/projects/**/*.jsonl` untuk backfill riwayat dan hitung biaya.

Subagent Claude (Task tool) tampil sebagai karakter kecil yang muncul di dekat induknya.

## 5. Lapisan provider

```ts
interface AgentProvider {
  id: string;                         // "claude-code" | "omp" | "codex" | ...
  capabilities: { resume: boolean; interrupt: boolean; approvals: boolean;
                  streaming: boolean; subagents: boolean; usage: boolean };
  start(opts: { cwd: string; prompt: string; model?: string; env?: Env }): Session;
  send(session: Session, msg: string): Promise<void>;
  interrupt(session: Session): Promise<void>;
  events(session: Session): AsyncIterable<MissionEvent>;   // sudah ternormalisasi
}
```

Adapter yang direncanakan:
| Adapter | Cara konek | Catatan |
|---|---|---|
| `claude-code` | SDK + hooks + MCP | fitur penuh |
| `omp` | *asumsi*: proses CLI/RPC mode dengan keluaran JSON-lines | **perlu konfirmasi** apa yang dimaksud OMP & mode integrasinya |
| `acp` (generik) | Agent Client Protocol (JSON-RPC via stdio) | satu adapter untuk banyak agent yang mendukung ACP |
| `cli-generic` | spawn proses + parser regex/JSONL yang bisa dikonfigurasi | fallback untuk agent apa pun |
| `openai-compat` | loop tool-use sendiri via HTTP API | untuk model/provider tanpa CLI agent |

`capabilities` menentukan UI: kalau provider tak punya approval, tombol “Setujui” disembunyikan; kalau tak punya streaming, karakter hanya punya state kasar (kerja/idle).

## 6. Visual 3D

**Scene**: ruang kontrol bergaya isometrik/low-poly. Satu meja kerja per agent, papan kanban besar di dinding (versi mini dari kanban asli), layar status di tengah.

**Mapping event → animasi** (state machine per karakter):
| Event | State | Visual |
|---|---|---|
| idle / tak ada session | `idle` | duduk santai, menyeruput kopi |
| `thinking` | `think` | tangan di dagu, gelembung “…” |
| tool Read/Grep/Glob | `read` | membolak-balik dokumen/menatap layar |
| tool Edit/Write | `type` | mengetik cepat, kode melayang |
| tool Bash | `run` | tekan tombol, lampu meja berkedip |
| tool Web* / MCP | `search` | memegang tablet/radar |
| `approval.requested` | `wait` | angkat tangan, ikon ❗ di atas kepala (klik → approve/deny) |
| `error` | `error` | kaget, asap kecil, meja merah |
| `session.ended` sukses | `celebrate` | lompat/mengangkat jempol, kartu pindah ke Review |
| subagent | `spawn` | karakter mini muncul dari meja induk |

**Interaksi**: klik karakter → Inspector (task aktif, transcript live, biaya, tombol interrupt/kirim pesan); drag kartu kanban ke karakter = assign; kamera: overview, follow-agent, top-down.

**Performa & aksesibilitas**: GLB ≤ 5k tri/karakter, instancing untuk subagent, target 60 fps di laptop biasa; mode **2D/“lite”** otomatis bila WebGL lemah atau `prefers-reduced-motion`; semua info 3D selalu tersedia juga sebagai teks (list agent).

Aset karakter: pilih dari galeri bawaan, impor VRM/GLB sendiri, atau generate (mis. lewat tool 3D generator) lalu rigging ke skeleton standar supaya animasi bisa dipakai ulang.

## 7. Kanban & task management

Kolom default: `Backlog → Ready → In Progress → Review → Done` (+ `Blocked`).

**Aturan otomatis** (bisa dimatikan per project):
- Assign agent + Start → `In Progress`, buat **git worktree + branch** `mc/<task-id>-<slug>` supaya agent paralel tidak bentrok.
- `approval.requested` / agent menunggu input → label `needs-you`, kartu naik ke atas kolom.
- Session selesai → `Review` + ringkasan diff & tes; tombol *Approve → merge / Request changes (kirim komentar kembali ke agent)*.
- Error berulang → `Blocked` dengan alasan.
- Task bisa dipecah: agent lewat MCP `add_subtask`, atau manual; dependensi antar-kartu (kartu `Ready` hanya bila dependensinya `Done`).

Fitur lain: WIP limit per kolom/agent, filter per agent/label, batas anggaran biaya per task, sinkron dua arah opsional dengan GitHub Issues (kolom ↔ label/Projects).

## 8. Keamanan

- Daemon hanya bind `127.0.0.1`; token acak untuk UI & hook receiver.
- Secret provider disimpan di keychain OS / env, tidak pernah dikirim ke UI atau ditulis ke event log.
- Default izin ketat: tool berisiko (Bash destruktif, tulis di luar worktree) selalu lewat Approval Broker; aturan allow bisa disimpan per agent.
- Agent bekerja di worktree terisolasi; merge hanya lewat aksi manusia di kolom Review.
- Remote access (kalau nanti ada) wajib TLS + auth, di luar lingkup v1.

## 9. Roadmap

| Fase | Isi | Selesai bila |
|---|---|---|
| 0 | Repo, daemon kerangka, skema event, SQLite | event dummy tersimpan & ter-stream |
| 1 | Kanban + task engine + worktree manager | task bisa dibuat, di-assign, branch terbentuk |
| 2 | Adapter Claude Code (SDK + hooks) | satu task jalan end-to-end, status live di kanban |
| 3 | Scene 3D v1: 1 ruangan, 1–3 karakter, 6 animasi inti | animasi cocok dengan event nyata |
| 4 | Approval Broker + Inspector + MCP kanban | approve dari UI, agent klaim kartu sendiri |
| 5 | Adapter OMP + ACP + cli-generic | dua provider berbeda berjalan bersamaan |
| 6 | Polish: biaya/token, replay sesi, tema, mode lite, impor VRM | siap rilis |

## 10. Risiko & pertanyaan terbuka

- **Apa itu “OMP” persisnya?** Kalau yang dimaksud agent tertentu (mis. *Oh My Pi*), saya perlu tahu mode integrasinya (RPC/JSON, plugin, dsb.). Sementara dirancang lewat `cli-generic`/`acp`.
- Hook vs SDK: sesi manual hanya *teramati* (tidak bisa dikendalikan penuh) — perlu dijelaskan jelas di UI.
- Beban 3D vs banyak agent: perlu batas jumlah karakter aktif dan LOD.
- Format event provider berubah antarversi: adapter harus diberi versi + tes kontrak dengan rekaman event.
- Biaya/token berbeda tiap provider: tampilkan sebagai “tidak tersedia” bukan nol.
