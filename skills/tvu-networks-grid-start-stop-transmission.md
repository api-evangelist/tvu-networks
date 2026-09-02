---
name: tvu-networks-grid-start-stop-transmission
description: Find a paired TVU transmitter, check it is available, start a live transmission and stop it.
api: TVU Networks Grid API
base_url: https://api.tvunetworks.com
operations:
  - getSession_250778482
  - getDeviceList_250778483
  - getAvailableDevice_250778507
  - getStatus_250778501
  - queryPairedToken_250778488
  - startLive_250778504
  - getRealTimeLiveParameters_250778509
  - stopLive_250778505
generated: '2026-09-01'
method: generated
source: openapi/tvu-networks-grid-api-openapi.yml
---

# Start and stop a TVU Grid transmission

Every operationId below is present in `openapi/tvu-networks-grid-api-openapi.yml`.

## Before you start

- Obtain a session with `POST /openapi/user/3.0/user/getSession` (`getSession_250778482`), or use `Authorization: Bearer <AppSecret>`.
- Application failures arrive as HTTP 200 with a non-`0x0` `errorCode`. See `errors/tvu-networks-problem-types.yml`.
- No idempotency mechanism exists on this surface. `startLive_250778504` is not safe to blind-retry — call `getStatus_250778501` first.

## Steps

1. **List devices.** `POST /openapi/device/3.0/device/getDeviceList` (`getDeviceList_250778483`).
2. **Filter to what can actually take a job.** `POST /openapi/device/3.0/device/getAvailableDevice` (`getAvailableDevice_250778507`).
3. **Check current state before acting.** `POST /openapi/device/3.0/device/getStatus` (`getStatus_250778501`). Treat "already live" as a stop condition, not as a reason to call `startLive` again.
4. **Confirm the pairing.** `GET /openapi/token/3.0/token/queryPairedToken` (`queryPairedToken_250778488`), or `queryPairedT_250778487` / `queryPairedR_250778486` for the transmitter and receiver sides. `queryPairedTGeoByRid_250778502` returns the transmitter's geolocation.
5. **Start.** `POST /openapi/device/3.0/device/startLive` (`startLive_250778504`).
6. **Monitor.** `GET /openapi/device/3.0/device/getRealTimeLiveParameters` (`getRealTimeLiveParameters_250778509`).
7. **Stop.** `POST /openapi/device/3.0/device/stopLive` (`stopLive_250778505`).

## Reversibility

`startLive_250778504` → `stopLive_250778505` is a complete, immediate reversal with no window. The same holds for the Anywhere push pair, `startPushLive_250778494` → `stopPushLive_250778495`. Pack tokens are reversible via `removePackToken_250778485`. Booking events created with `addEvent_250778496` are removed by `deleteEvent_250778497` — a hard delete with no published retention window.
