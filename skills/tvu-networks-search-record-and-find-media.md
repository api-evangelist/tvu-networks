---
name: tvu-networks-search-record-and-find-media
description: Trigger a recording with speech transcription and face detection, then find the resulting media and subscribe to metadata webhooks.
api: TVU Networks Search API
base_url: https://api.tvunetworks.com
operations:
  - Create_a_new_slug_and_trigger_recording_450372672
  - Update_an_existing_slug_and_its_recording_configuration_450372673
  - register_webhook_414798733
  - delete_slug_351711929
generated: '2026-09-01'
method: generated
source: openapi/tvu-networks-search-api-openapi.yml, openapi/tvu-networks-uncategorized-openapi.yml, https://docs.tvunetworks.cn/doc-8184935
---

# Record media with AI metadata and find it again

Grounded in `openapi/tvu-networks-search-api-openapi.yml` and `openapi/tvu-networks-uncategorized-openapi.yml`. The ingest sequence follows TVU's own "Ingest Integration Workflow" page.

## Before you start

- `Authorization: Bearer <AppSecret>` is accepted on the Search surface; `SID` also works.
- Failures are HTTP 200 with a non-`0x0` `errorCode`.
- Pagination on this surface is inconsistent: some operations take `pageNum`/`pageSize`, others take `Offset`. No response declares a total count, so you cannot tell from the contract when you have reached the last page — stop when a page comes back short.

## Steps

1. **Create the source identity.** `POST /mediahub/api/v2/sourceObject` (`createSourceObject`, in the MediaHub spec) to represent the input — a TVU Pack, an external URL, an S3 object. Keep `result.objectId`.
2. **Create a slug and start recording.** `POST /search/v2/media/createSlugRecord` (`Create_a_new_slug_and_trigger_recording_450372672`). Enable the AI passes in the request body: `"speechTranscription": 1` for STT, `"faceDetection": 1` for vision.
3. **Adjust while running.** `POST /search/v2/media/updateSlugRecord` (`Update_an_existing_slug_and_its_recording_configuration_450372673`).
4. **Subscribe to completion instead of polling.** `POST /api/metadataproto/MetadataUploader.RegisterWebhook` (`register_webhook_414798733`) with `{webhookURL, operation: "register", sourceObject: [...]}`. An empty `sourceObject` list means all source objects. TVU then POSTs `{status, sourceObjectId, mediaObjectId, previewMediaPath, highResMediaPath, thumbnailPath, mediaFileType}` to your URL.
   - **Verify nothing.** TVU publishes no signature, shared secret or timestamp on webhook delivery, and no retry policy. Treat the callback as an untrusted hint and re-read state over the API before acting on it. See `asyncapi/tvu-networks-webhooks.yml`.
5. **Search.** Use the cross-type media content search operations in the Search spec to locate media by object, person, word or time range.
6. **Clean up.** `POST /search/v2/media/deleteSlug` (`delete_slug_351711929`).

## Reversibility

The recording slug is reversible: `delete_slug_351711929` removes it. Webhook registration is reversible by re-calling `register_webhook_414798733` with `operation: "unregister"`. Neither carries a published window. Deleted media has no restore path.
