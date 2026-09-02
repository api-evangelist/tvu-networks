---
name: tvu-networks-export-media-idempotently
description: Submit a TVU media export or thumbnail job safely, using the one idempotency mechanism this API publishes, and read the real result out of an HTTP 200.
api: TVU Networks Export
base_url: https://api.tvunetworks.com
operations:
  - estimateVideoSize
  - publicExport
  - publicLongExport
  - exportAudioV2
  - cutJsonV2
  - mergeVideo
generated: '2026-09-01'
method: generated
source: openapi/tvu-networks-export-openapi.yml + https://docs.tvunetworks.cn/doc-5808220
---

# Export TVU media without creating duplicate jobs

Grounded in `openapi/tvu-networks-export-openapi.yml` and TVU's published ERROR CODE table.

## Why this skill exists

The export surface is the **only** place in the whole TVU public API where idempotency is documented. Every other write surface will happily execute twice on a retry. Use it.

## Steps

1. **Size the job first.** `POST /estimate-video-size` (`estimateVideoSize`). Four of TVU's published error codes are quota rejections (1031, 1032, 1046, 1052) and none of them states the limit it enforces, so estimating first is the only way to avoid discovering a ceiling by hitting it.
2. **Generate a stable `uuid` for this logical export** — derive it from your own job identity, not from a random per-attempt value. This is the idempotency key.
3. **Submit.** `POST /export` (`publicExport`), or `publicLongExport` for long-form, `exportAudioV2` for audio, `cutJsonV2` and `mergeVideo` for edit operations. Include:
   - `uuid` — the idempotency key. Re-submitting the same `uuid` returns the existing job rather than creating a second one.
   - `callbackUrl` — the webhook TVU POSTs to when the job completes.
4. **Read the outcome from the body, not the status.** The HTTP status will be 200. Branch on `errorCode`:
   - `1000` job created · `1001` in progress · `1005` done · `1100` **the file already existed and can be downloaded directly** — this is the idempotent hit, and it is a success, not an error.
   - `1010` missing source file · `1018` key parameters missing · `1090` source ts file not exist — client errors; do not retry unchanged.
   - `1031`, `1032`, `1046`, `1052` — quota. Reduce duration or free space; retrying identically will fail identically.
   - `1099` — "Media processing encountered an exception. Please retry." The one code TVU explicitly marks retryable. Reuse the same `uuid`.
   - Full table: `errors/tvu-networks-problem-types.yml`.
5. **Wait for the callback**, or poll. There is no job-status endpoint separate from re-submitting the same `uuid`.

## Reversibility

**There is none.** No cancel or abort operation exists for an in-progress export. Once `1001` comes back the job runs to completion. The `uuid` prevents a *duplicate*; it does not let you take an export back. Size the job before you submit it.
