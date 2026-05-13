---
title: "`media` & `stt` — Attachments and Speech-to-Text"
type: module
sources:
  - ../src/media.rs
  - ../src/stt.rs
  - ../docs/stt.md
  - ../docs/sendimages.md
  - ../docs/sendfiles.md
related: [adapter, ../adapters/discord]
updated: 2026-05-12
---

`src/media.rs` (432 行) 處理圖片下載/縮圖/壓縮與檔案附件下載;`src/stt.rs`
(354 行) 是 OpenAI 相容的 Whisper API 客戶端,把 Discord 語音訊息轉成文字後
餵給 agent。兩者都是「**進入 ACP `ContentBlock` 之前**」的最後一道前處理。

`media.rs` (432 LOC) downloads, resizes, and compresses images/attachments;
`stt.rs` (354 LOC) is an OpenAI-compatible Whisper API client. Both run
**before** content reaches the ACP `ContentBlock` stage.

---

## `media.rs`

### Image pipeline / 圖片管線

1. **Download** the attachment from the platform's CDN (with `reqwest`).
2. **Decode** via the `image` crate (jpeg / png / gif / webp features).
3. **Resize** if larger than the per-platform max (broker-side cap to keep
   ACP `ContentBlock::Image` base64 payloads reasonable).
4. **Re-encode** as JPEG (default) or keep original format for transparency-
   sensitive cases.
5. **Base64-encode** and produce a `ContentBlock::Image { media_type, data }`.

### File pipeline / 檔案管線

For non-image attachments (text-ish files within a size cap):

1. Download.
2. Read as UTF-8 if possible; otherwise base64-encode and label by extension.
3. Inline as a `ContentBlock::Text` with a fenced code block prefix
   identifying the filename and type.

Hard binary files (large PDFs, archives) are not inlined; the broker logs a
note and skips, since ACP `ContentBlock` does not have a generic file type.

### Helpers / 工具函式

- Probing image dimensions before full decode (saves memory on giant uploads).
- Detecting MIME by magic bytes (don't trust the platform's reported type).

---

## `stt.rs`

### Provider model / 提供者模型

OpenAI-compatible by default. `SttConfig`:

```toml
[stt]
enabled         = true
api_key         = "${STT_API_KEY}"
model           = "whisper-large-v3-turbo"          # default
base_url        = "https://api.groq.com/openai/v1"  # default → Groq
echo_transcript = false                              # echo to thread before agent dispatch
```

Three known good providers:

| Provider | `base_url` | Notes |
|---|---|---|
| Groq (default) | `https://api.groq.com/openai/v1` | fastest; `whisper-large-v3-turbo` |
| OpenAI | `https://api.openai.com/v1` | `whisper-1` |
| Local Whisper server | `http://localhost:8000/v1` | self-hosted; no API key |

All three speak the same `/audio/transcriptions` endpoint.

### Discord voice messages / Discord 語音訊息

When a Discord message has a voice-message attachment:

1. `media.rs` downloads the Opus audio file.
2. `stt.rs` posts it to `<base_url>/audio/transcriptions`.
3. The text comes back.
4. If `echo_transcript = true`, the broker posts the transcript to the
   thread (no @mentions) so the user can sanity-check accuracy.
5. The transcript is then **prepended** to the user's content blocks before
   the prompt goes to the agent.

### Why echo / 為什麼要 echo

STT accuracy varies wildly across accents, technical jargon, and background
noise. Echoing the transcript lets users catch a misrecognition before the
agent burns tokens on the wrong question.

---

## Where they slot in / 在管線中的位置

Both run during gate ⑦'s "build content blocks" step (see
[architecture/message-flow](../architecture/message-flow.md)). Output is one
or more `ContentBlock`s appended to the dispatcher's per-thread/lane buffer.

---

## User-facing docs / 對應使用者文件

- `../docs/sendimages.md` — how to send images.
- `../docs/sendfiles.md` — how to send files.
- `../docs/stt.md` — STT setup.
