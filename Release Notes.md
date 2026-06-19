# MIDAS API v2.0 — Release Notes

**Release Date:** June 22, 2026

**Version:** 2.0.0

**Previous Version:** 1.0

**Contact:** <midas@energy.ca.gov>

---

## Overview

MIDAS v2.0 is a major upgrade to the Market Informed Demand Automation Server. This release delivers substantial improvements to performance and reliability, simplifies data access for GET users, introduces programmatic upload tracking, and consolidates GHG and Flex Alert signals into streamlined RINs. LSE uploaders must re-register and request access under the new workflow before uploading after June 22, 2026.

---

## Breaking Changes

These changes require action from existing integrations. See the [Transition Guide](Transition-Guide.md) for detailed migration steps.

### BC-1 — GHG Emissions Unit Change: `kg/kWh CO2` → `g/kWh CO2`

All SGIP GHG emissions values are now reported in **grams per kilowatt-hour** (`g/kWh CO2`), replacing the previous unit of kilograms per kilowatt-hour (`kg/kWh CO2`). Numeric values are 1,000× larger than they were in v1.0.

- **Affected:** Any application that reads `Value` from SGIP GHG RINs.
- **Action required:** Multiply stored thresholds, comparisons, and chart axes by 1,000. Update unit labels from `kg/kWh CO2` to `g/kWh CO2`.

### BC-2 — SGIP GHG RIN Consolidation (33 → 11 RINs)

The 33 SGIP GHG RINs (11 regions × 3 types: SGRT/SGFC/SGHT) have been replaced by 11 MOER RINs — one per region.

| Deprecated Pattern | Replacement |
| --- | --- |
| `USCA-SGIP-SGRT-{REGION}` | `USCA-SGIP-MOER-{REGION}` with `QueryType=realtime` |
| `USCA-SGIP-SGFC-{REGION}` | `USCA-SGIP-MOER-{REGION}` with `QueryType=realtime` |
| `USCA-SGIP-SGHT-{REGION}` | `USCA-SGIP-MOER-{REGION}` with `QueryType=alldata` |

### BC-3 — Flex Alert RIN Consolidation (3 → 1 RIN)

Three Flex Alert RINs replaced by one unified RIN.

| Deprecated RIN | Replacement |
| --- | --- |
| `USCA-FLEX-FXRT-0000` | `USCA-FLEX-ALRT-0000` with `QueryType=realtime` |
| `USCA-FLEX-FXFC-0000` | `USCA-FLEX-ALRT-0000` with `QueryType=realtime` |
| `USCA-FLEX-FXHT-0000` | `USCA-FLEX-ALRT-0000` with `QueryType=alldata` |

### BC-4 — RINList Response Structure Changed

The `GET /api/valuedata?SignalType=N` response is now a keyed object instead of a bare array.

**v1.0:**

```json
[{ "RateID": "...", "SignalType": "Rates", "Description": "..." }]
```

**v2.0:**

```json
{ "Rates": [{ "RateID": "...", "SignalType": "Electricity Rates", "Description": "...", "LastUpdated": "DateTime" }] }
```

### BC-5 — Upload Response Changed: 200 → 202, New `jobID` Field

`POST /api/valuedata` now returns `HTTP 202 Accepted` (previously `200 OK`) with a `jobID` field.

### BC-6 — LSE Re-Registration Required

All existing LSE upload accounts must re-register and go through the new upload-access request workflow. Old credentials will not grant upload access on the new system.

### BC-7 — Removed Endpoints

The following endpoints are removed with no replacement:

- `GET /api/historicallist` (`HistoricalRINList`)
- `GET /api/Holiday` (`Holiday`)
- `TimeZone` lookup table (via `GET /api/valuedata?LookupTable=TimeZone`)

---

## New Features

### NF-1 — Jobs Endpoint for Upload Tracking

Two new endpoints for tracking uploads:

- `GET /api/jobs` — Lists all upload jobs from the last 7 days for the authenticated user, newest first.
- `GET /api/jobs/{jobID}` — Returns full processing detail for one upload, including per-RIN validation issues.

Job statuses: `PROCESSING`, `COMPLETE`, `VALIDATION_FAILED`.

Job IDs are UUID v7 — time-sortable and guaranteed unique.

### NF-2 — Self-Service Upload Access Request

LSEs can now request upload authorization directly through the API:

- `POST /api/uploadaccess/request` — Submits a request for upload access for a specific energy/distribution code pair.

A CEC admin approves requests via `POST /admin/approvelseaccess`.  

Note: Accounts can hold multiple energy codes and distribution codes simultaneously.

### NF-3 — WattTime and CAISO Health Endpoints

New health and monitoring endpoints for operational visibility:

- `/watttime/health`, `/watttime/health/detailed`, `/watttime/health/comprehensive`
- `/watttime/metrics`, `/watttime/status`
- `/caiso/health`, `/caiso/status`

### NF-4 — System Root and Health Endpoints

- `GET /` — API root with version and navigation links.
- `GET /health` — Standard health check for load balancer and monitoring integration.

### NF-6 — Email / Password Self-Service

New programmatic flows for account maintenance:

- `POST /api/forgotpassword` — Initiates password reset via email token.
- `POST /api/confirmpasswordreset` — Confirms reset with a one-time token.
- `POST /api/resendverification` — Resends the email verification code.

---

## Improvements

### IM-1 — Infrastructure Upgrade

MIDAS v2.0 is running on upgraded AWS compute and storage. The application layer has been migrated to a newer runtime with improved autoscaling, higher throughput, and reduced latency. Additional system-level health-check endpoints are available for monitoring integrations.

### IM-2 — No Registration Required for GET Requests

All public read endpoints (`GET /api/valuedata` with `SignalType` or `ID+QueryType` or `LookupTable`) are now unauthenticated. Token-based access is only required for uploads and jobs endpoint.

### IM-3 — Extended Token Lifetime

Tokens are now valid for **3,600 seconds** (1 hour), up from 600 seconds (10 minutes). This significantly reduces the overhead of re-authenticating during long upload sessions.

### IM-4 — Multi-Code LSE Accounts

A single LSE account may now be authorized for multiple distribution codes and multiple energy codes, eliminating the need for multiple accounts at larger utilities and CCAs.

### IM-5 — Two-Stage Upload Validation

Uploads now receive immediate feedback on basic structural validity (stage 1, synchronous) followed by asynchronous deep validation (stage 2). The Jobs endpoint surfaces the results of both stages.

### IM-6 — Data Retention

- 2 years: Active, accessible via API
- Years 2–7: Cold storage archive (contact <midas@energy.ca.gov> for access) - CEC is planning to automate this in future upgrades
- Older than 7 years: On request from CEC

### IM-7 — Interval Validation Enforced

Uploads are validated against the interval assigned to a RIN at first upload. Disallowed intervals (anything other than 5 minutes, 15 minutes, or 1 hour) are rejected. Cross-unit consistency within an upload is also enforced.

### IM-8 — Improved Error Description

This upgrade returns better human readable detailed messages for users when an error occurs.

---

## Deprecations

The following are deprecated as of v2.0 and will be removed in a future version.

- All `USCA-SGIP-SGRT-*` RINs
- All `USCA-SGIP-SGFC-*` RINs
- All `USCA-SGIP-SGHT-*` RINs
- `USCA-FLEX-FXRT-0000`
- `USCA-FLEX-FXFC-0000`
- `USCA-FLEX-FXHT-0000`

---

## Support

Contact the MIDAS team at **<midas@energy.ca.gov>** for questions, issue reports, or to request access to archived data older than 2 years.

---

*California Energy Commission | MIDAS v2.0 | Released June 22, 2026*
