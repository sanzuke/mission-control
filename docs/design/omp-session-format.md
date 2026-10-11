# Format sesi OMP (Oh My Pi) - hasil analisis sampel

> Dokumen living. Sumber: `docs/sample-omp.jsonl` (omp v18.8.9, sesi format `version: 3`, 63 baris, 1 sesi, 11 giliran asisten, 22 panggilan tool). Satu sampel saja: apa yang tidak ada di sampel ditandai **belum terverifikasi**.

## 1. Struktur berkas

Satu objek JSON per baris. Dua baris pertama adalah header:

| Baris | `type` | Isi penting |
|---|---|---|
| 1 | `title` | `{v:1, title, source:"auto", updatedAt, pad}`. Panjang baris **tetap 256 byte** karena ada field `pad` berisi spasi |
| 2 | `session` | `{version:3, id, timestamp, cwd, title, titleSource}` |

**Konsekuensi penting:** baris 1 ditulis ulang di tempat (judul sesi berubah tanpa mengubah ukuran berkas). Watcher tidak boleh mengira berkas murni append-only: baca header terpisah (ulangi baca 256 byte pertama saat `mtime` berubah) dan lakukan tailing dari offset terakhir untuk sisanya. Jika ukuran berkas mengecil, mulai ulang dari awal.

Baris berikutnya adalah entri dengan `id` (8 hex), `parentId`, `timestamp` (ISO). `parentId` membentuk pohon (OMP mendukung fork/branch); di sampel rantainya linear. Watcher mengikuti urutan berkas dan menyimpan `parentId` untuk fitur cabang kelak.

## 2. Jenis entri

| `type` | Keterangan | Dipakai untuk |
|---|---|---|
| `model_change` | `model: "anthropic/claude-opus-5-5"`, `resolvedModelIsFallback` | label model di Inspector |
| `thinking_level_change` | `thinkingLevel: "high"` | label |
| `message` role `user` | `content:[{type:"text",text}]`, `attribution:"user"` | awal giliran |
| `message` role `assistant` | blok `thinking` / `text` / `toolCall`; `stopReason`; `usage`; `provider`; `model`; `duration`; `ttft`; `completedAt` | aktivitas, biaya |
| `custom` `customType:"tool_execution_start"` | `data:{toolCallId, toolName, startedAt, args, intent}` | **mulai tool + deskripsi manusiawi** |
| `message` role `toolResult` | `toolCallId, toolName, content, isError, details` | akhir tool |
| `title_change` | judul otomatis baru | judul kartu/sesi |
| `credential_pin` | `provider`, `hash` | **abaikan** (jangan simpan/ tampilkan) |

Blok `toolCall` di pesan asisten: `{type:"toolCall", id, name, arguments, intent}`; `id` sama dengan `toolCallId` pada `tool_execution_start` dan `toolResult`.

Blok `thinking` isinya kosong (hanya `thinkingSignature`): isi pemikiran **tidak terbaca** dari berkas.

## 3. Aturan turunan aktivitas (`AgentActivity`)

| Kejadian terakhir | Aktivitas | Catatan |
|---|---|---|
| `message` user | `thinking` | giliran dimulai |
| `assistant` dengan `stopReason:"toolUse"` | `tool_use` | blok `toolCall` menyebut apa yang akan jalan |
| `tool_execution_start` | `tool_use` + `intent` | `intent` = teks siap tampil, mis. "Reading dashboard memory index" |
| `toolResult` (semua panggilan terbuka sudah selesai) | `thinking` | menunggu respons model berikutnya |
| `toolResult` sebagian | tetap `tool_use` | panggilan paralel (lihat 4) |
| `assistant` dengan `stopReason:"stop"` | `idle` / `waiting_input` | **batas giliran yang pasti**, tidak perlu timeout 5 dtk |
| tidak ada entri baru > N detik saat ada panggilan terbuka | `waiting_permission` atau tool lambat | **belum terverifikasi**, lihat 6 |

Pemetaan nama tool → animasi (sampel memuat `read, write, bash, web_search, glob`; help menyebut juga `edit, grep, lsp, task, todo, browser, notebook, python`):

| Tool | Animasi |
|---|---|
| `read`, `glob`, `grep`, `lsp` | membaca |
| `write`, `edit`, `notebook` | mengetik |
| `bash`, `python` | menjalankan |
| `web_search`, `browser` | mencari |
| `task` | memunculkan subagent (belum terverifikasi) |

**Jangan andalkan nama tool saja.** Di sampel, tool `write` dipakai dengan `intent` seperti "Listing baton projects" dan "Searching baton memory", dan argumen `read` memakai skema `xd://mcp__...` (kemungkinan jalur tool MCP/eksternal). Aturan: animasi dari nama tool, teks gelembung dari `intent`, dan skema `xd://` ditandai sebagai "tool eksternal". Tool yang tidak dikenal → animasi generik "bekerja".

## 4. Perilaku penting

- **Panggilan paralel:** dua `tool_execution_start` berurutan (baris 9-10) sebelum dua `toolResult`; pasangkan dengan `toolCallId`, jangan dengan urutan.
- **Entri asisten ditulis setelah respons selesai** (`completedAt`); proses berpikir/streaming tidak terlihat. Resolusi waktu nyata = per pesan, bukan per token. Untuk streaming halus perlu jalur `acp`/`rpc`.
- **`isError:true` di `toolResult`** (3 dari 22) = kegagalan satu tool, bukan kegagalan sesi. Tampilkan sebagai percikan kecil, bukan status `error` agent.
- **Sesi dibuka ≠ bekerja:** `session.timestamp` 04:37, pesan user pertama 05:41. Status `idle` sampai ada pesan user.
- Biaya langsung terbaca: `usage.{input,output,cacheRead,cacheWrite,totalTokens,cost.total}` per pesan asisten (total sampel: **US$ 0,717** untuk 11 giliran) + `duration`, `ttft`, `model`, `provider`. Menu Usage tidak perlu memanggil `omp usage` untuk sesi yang terlihat.
- `cwd` di header menentukan proyek/ruangan; `id` sesi = `providerSessionId`.

## 5. Pemetaan ke `MissionEvent`

| Sumber OMP | `MissionEvent` |
|---|---|
| `session` | `session.started` {providerSessionId, cwd} |
| `title_change`/`title` | `session.titled` |
| `model_change` | `session.model` |
| `message` user | `message` (role user) + `activity.changed`→thinking |
| `message` assistant | `message`/`usage` (+ `activity.changed`) |
| `tool_execution_start` | `tool.start` {toolCallId, name, intent, args-ringkas} |
| `toolResult` | `tool.end` {toolCallId, ok, ringkasan-terpotong} |
| `assistant` `stop` | `activity.changed`→waiting_input |

Konten (`args`, `toolResult.content`, teks) dapat memuat secret/path pribadi: simpan terpotong dan tandai sensitif; **jangan** teruskan `credential_pin`, `thinkingSignature`, `requestControls`.

## 6. Belum terverifikasi (butuh sampel tambahan)

1. **Izin tool**: sesi dengan `--approval-mode always-ask` - entri apa yang muncul saat menunggu persetujuan? (menentukan `waiting_permission`).
2. **Subagent** (`task`): apakah ada entri `custom` lain, berkas terpisah, atau rujukan `parentSession`? (`omp anonymize` menyebut "session and its subagents").
3. **Edit**: bentuk `args` tool `edit` (untuk menampilkan nama berkas yang diubah).
4. **Cabang/fork** dan `compaction` (`contextSnapshot.compactionEpoch`) - bagaimana dicatat.
5. **Jalur kendali** `omp acp` dan `--mode json/rpc`: tangkap satu prompt dengan satu panggilan tool, untuk golden file adapter kendali.
6. Berkas sesi yang sedang aktif ditulis per baris atau per batch? (menentukan interval polling watcher).

## 7. Rencana tes

Salin sampel menjadi fixture `tests/fixtures/omp/session-v3.jsonl` (setelah redaksi path pribadi/isi memori) dan tulis tes kontrak parser: jumlah event, pasangan `toolCallId`, penurunan aktivitas di tiap baris, ketahanan terhadap baris terpotong di akhir berkas dan tipe entri tak dikenal (dicatat, bukan crash).
