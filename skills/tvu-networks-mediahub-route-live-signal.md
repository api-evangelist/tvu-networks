---
name: tvu-networks-mediahub-route-live-signal
description: Route a live video source through TVU MediaHub to one or more encoded destinations, then take it down cleanly.
api: TVU Networks MediaHub API
base_url: https://api.tvunetworks.com
operations:
  - createEncodingProfile
  - queryEncodingProfileDict
  - createOutput
  - outputPairedWithEncodingProfile_1
  - createProject
  - createSourceObject
  - connect
  - queryStatus
  - disconnect
generated: '2026-09-01'
method: generated
source: openapi/tvu-networks-mediahub-api-openapi.yml + https://docs.tvunetworks.cn/folder-47022365
---

# Route a live signal through TVU MediaHub

Every operationId below is present in `openapi/tvu-networks-mediahub-api-openapi.yml`. The sequence is TVU's own "Complete Workflow" from the MediaHub folder documentation.

## Before you start

- Authenticate with `Authorization: Bearer <AppSecret>` or the `SID` header. MediaHub accepts both; TVU's docs say to disable whichever one you are not using, because sending both is a documented cause of failure.
- Read `errors/tvu-networks-problem-types.yml` first. **This API returns HTTP 200 on failure.** Branch on `errorCode == "0x0"`, never on the status code.
- There is no idempotency key on any of these operations. A retried `connect` can double-fire. Check `queryStatus` before retrying.

## Steps

1. **Discover the encoding vocabulary.** `GET /urcaller/api/v2/encodingProfile/dict` (`queryEncodingProfileDict`) returns the allowed values for every encoding-profile attribute. Do this first — the profile body has many constrained fields and the dict is the only published source of legal values.
2. **Create an encoding profile.** `POST /urcaller/api/v2/encodingProfile` (`createEncodingProfile`). Keep `result.id`.
3. **Create an output.** `POST /urcaller/api/v2/output` (`createOutput`). If MediaHub is running as a listener you may omit `url` and the API returns one. Keep `outputId`.
   - Output `url` grammar (RTMP, RTSP, HLS, SRT, UDP, NDI, PROMPEG, SDI, GRID) is published at https://docs.tvunetworks.cn/doc-5661551. Use it verbatim; a malformed scheme is rejected as a generic error.
4. **Pair the profile to the output.** `POST /urcaller/api/v2/output/encodingProfile/pair` (`outputPairedWithEncodingProfile_1`). Each pairing returns an `outputEncodingProfilePairId` — this is what TVU's UI calls a *destination*.
5. **Create a project.** `POST /mediahub/api/v2/project` (`createProject`), passing one or more `outputEncodingProfilePairId` values. Keep `projectObjectId`.
6. **Create a source object.** `POST /mediahub/api/v2/sourceObject` (`createSourceObject`) with `sourceType`, `sourceName`, `region` and `url`. Keep `result.objectId`.
7. **Go live.** `POST /mediahub/api/v2/route/connect` (`connect`) binding the source object to the project.
8. **Confirm.** `GET /mediahub/api/v2/project/status` (`queryStatus`).
9. **Take it down.** `POST /mediahub/api/v2/route/disconnect` (`disconnect`). To stop a single destination instead of the whole route, use `destinationStopLive`; to stop just one output leg, `singleOutputDisableStreaming`.

## Reversibility

`connect` is fully reversible by `disconnect`, immediately and with no window — this is signal routing, not a transaction. Everything you created in steps 2–6 is reversible only by a **hard delete** (`deleteEncodingProfile`, `deleteOutput`, `deletePairedOfOutputEncodingProfile`, `deleteProject`, `deleteSourceObjectInfo`). TVU publishes no restore, undelete or trash operation, and no retention window. Delete only what you created.
