# Glossary Domain — Mission Control

> Tujuan: istilah yang dipakai stakeholder dan istilah di kode harus bisa
> dipetakan satu sama lain. Ketidakcocokan di sini adalah sumber bug yang
> tidak kelihatan sampai terlambat.

| Istilah (domain) | Di kode | Arti | Catatan |
|---|---|---|---|
| Agent | `Agent` | Identitas persisten AI coding (provider, model, persona, avatar) | Tampil sebagai karakter 3D |
| Session | `Session` | Satu run agent untuk satu task | Punya `sessionId` provider |
| Task / Kartu | `Task` | Unit kerja di kanban | 1 task = 1 git worktree |
| Provider | `AgentProvider` | Backend agent (Claude Code, OMP, Codex, ...) | Masuk lewat adapter |
| Event | `MissionEvent` | Kejadian ternormalisasi dari provider | Penggerak animasi & kanban |
| Approval | `ApprovalBroker` | Izin manusia untuk tool berisiko | Dari hook `PreToolUse` / `canUseTool` |

## Istilah yang gampang tertukar

| | Bukan | Bedanya |
|---|---|---|
| | | |
