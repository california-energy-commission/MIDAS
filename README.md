# Market Informed Demand Automation Server (MIDAS) — v2.0 Documentation 

### California Energy Commission | midas@energy.ca.gov  

MIDAS API Swagger Documentation: [MIDAS API Swagger](https://i7k3zmg44f.us-west-2.awsapprunner.com/docs#/)

---

> **Release Notice — June 22, 2026**  
> MIDAS v2.0 is live as of June 22, 2026. If you are migrating from v1.0, please read the [Transition Guide](transition-guide.md) before making any API calls. The base URL for the production system is **https://midasapi.energy.ca.gov**. All test-environment URLs referenced in pre-release materials will not be active after July, 2026.

--- 
![](https://img.shields.io/badge/Note-MIDAS%201.0%20docs%20preserved-red)  
> MIDAS 1.0 documentation is preserved at the [MIDAS v1.0 Documentation](https://github.com/california-energy-commission/MIDAS/releases/tag/v1.0) on the official CEC repository.
--- 

## Table of Contents

1. [Introduction](#1-introduction)
2. [What's New in v2.0](#2-whats-new-in-v20)
3. [Database Structure](#3-database-structure)
4. [Authentication](#4-authentication)
5. [GET Requests — Retrieving Data](#5-get-requests--retrieving-data)
6. [POST Requests — Uploading Data (LSE Users)](#6-post-requests--uploading-data-lse-users)
7. [Python Examples](#7-Python-Examples) 
8. [Jobs Endpoint — Tracking Upload Status](#8-jobs-endpoint--tracking-upload-status)
9. [System Health Endpoints](#9-system-health-endpoints)
10. [Error Handling](#10-error-handling)
11. [Rate Limits](#11-rate-limits)
12. [Appendices](#appendices)

---

## 1. Introduction

The California Energy Commission’s (CEC) Market Informed Demand Automation Server (MIDAS) is a database and application programming interface (API) that provides access to current, future, and historic time-varying rates, greenhouse gas (GHG) emissions associated with electrical generation, and California Flex Alert Signals. The database is populated by utilities and community choice aggregators (CCAs), WattTime’s Self-Generation Incentive Program (SGIP) marginal GHG emissions API, the California Independent System Operator (California ISO), and other entities that are registered with the MIDAS system.

MIDAS is designed to provide energy users with the electricity price information they need to optimize when they use energy. While it would be useful for electricity users to be able to use the data from MIDAS to try to estimate electricity bill totals, some billing structures such as tiered rates make this impossible without private customer-specific data. MIDAS does not, and will not, contain any private information. The only non-public data in MIDAS is login information for data uploaders.

MIDAS is accessible through a public API at <https://midasapi.energy.ca.gov> in two standard machine-readable formats: JavaScript Object Notation (JSON), and extensible markup language (XML). MIDAS is publicly accessible, allowing all registered users to query and download information by interacting with the MIDAS API. CEC strongly encourages load serving entity (LSE) users have programming skills and software to effectively upload and maintain rate information stored in the database. Registration is a simple process available through the API. Non-LSE users should be able to retrieve information stored in MIDAS without extensive programming skills. Retrieving MIDAS-hosted data can be easily done through the code examples provided or through a user’s own code. For instructions on accessing the MIDAS database, see [GET Requests — Retrieving Data](#5-get-requests--retrieving-data) and/or [Python Examples - Appendix g](appendix-g.md).

MIDAS was developed to support the CEC's [Load Management Standards](https://www.energy.ca.gov/programs-and-topics/topics/load-flexibility/load-management-standards). The full text of the standards is available from [Westlaw](https://govt.westlaw.com/calregs/Browse/Home/California/CaliforniaCodeofRegulations?guid=ID6B950105CCE11EC9220000D3A7C4BC3).

### Who Uses MIDAS?

| User Type | What They Do |
|-----------|--------------|
| **Read-only users** | Query rates, GHG signals, and Flex Alert data. No registration required as of v2.0. |
| **LSE uploaders** | Utilities, CCAs and other organizations that POST rate data. Must register, verify email, and request upload access from MIDAS team through the new access-request API endpoint. |
| **Integrators / automation providers** | Developers building load management systems, OpenADR clients, or analytics platforms on top of MIDAS data. |

**Base URL:** `https://midasapi.energy.ca.gov`

**Supported formats:** JSON and XML (XML payloads are accepted for uploads and are converted internally)

**API version:** 2.0 

### Security

The MIDAS API and database are protected by the CEC firewall and data throttling to prevent Distributed Denial of Service (DDoS) attacks (see [Appendix D](appendix-d.md)). If an error occurs after an API call, the program will send a notification for CEC information technology (IT) staff to fix the issue.

---

## 2. What's New in v2.0

See also: [Release Notes](release-notes.md) for the full changelog and [Transition Guide](transition-guide.md) for migration steps.

### For GET Users (Data Consumers)

- **No registration required** to query data. All `GET /api/valuedata` calls are now publicly accessible without a token.
- **Realtime query window:** 72 hours starting at 00:00:00 Pacific on the day of the request. The datetime values are returned in UTC.
- **Alldata query window:** 90 days ending at 23:59:59 Pacific on Day+2 of the request date. The datetime values are returned in UTC.
- **Unified SGIP GHG RINs:** 33 RINs (SGRT/SGFC/SGHT per region) consolidated into 11 MOER RINs — one per grid region.
- **Unified Flex Alert RIN:** Three RINs (FXRT/FXFC/FXHT) replaced by a single `USCA-FLEX-ALRT-0000` RIN.
- **GHG unit change:** Emissions values now reported in `g/kWh CO2` (previously `kg/kWh CO2`). Multiply old values by 1,000.
- **Updated RINList response format:** Responses are now wrapped in a top-level keyed object.
- **Removed endpoints:** `HistoricalRINList`, `Holiday`, `TimeZone` lookup table.
- **Data retention:** 2 years of active data directly accessible via API; 5 years in cold storage archive; data older than 7 years available on request at midas@energy.ca.gov.

### For POST Users (LSE Uploaders)

- **Re-registration required.** All existing LSE accounts must re-register and re-verify email before uploading.
- **New upload access workflow:** After registration, submit a formal request via `POST /api/uploadaccess/request`. A CEC admin will approve the request and assign authorized distribution and energy codes.
- **Multiple codes per account:** A single account may now hold multiple distribution codes and multiple energy codes.
- **Two-stage validation:** The API returns `HTTP 202 Accepted` with a `jobID` after initial validation passes. Final validation (data quality, regulatory checks) runs asynchronously. Check status at `GET /api/jobs/{jobID}`.
- **Extended token lifetime:** Tokens are now valid for 3,600 seconds.
- **Interval enforcement:** Allowed upload intervals are 1 hour, 15 minutes, and 5 minutes only. The interval is assigned to a RIN on first upload and cannot change. All units in an upload must use the same interval and the same count of data points.
---

## 3. Database Structure

The MIDAS database supports retrieval of electric utility price schedules, California Flex Alert signals, and marginal GHG emissions from electrical generation. Flex Alerts and GHG emissions values – both forecasted and real-time - are continually retrieved from the California ISO Flex Alert site and WattTime’s SGIP API respectively. GHG realtime and forecasted values are cached, or temporarily stored, within MIDAS until new values are available while Flex Alerts are passed through MIDAS from the California ISO website, as a user queries MIDAS. A record of previously active Flex Alerts and historic GHG emissions are stored in the database.

Pursuant to the California Load Management Standards, the state’s largest utilities and CCAs are responsible for populating the MIDAS database with all time-varying rate information and values offered to customers, including the pilot rates. For upload examples, please see [Appendix A](appendix-a.md).

The primary lookup identification (ID) for the MIDAS database is a compound key comprised of six individual fields that make up a standardized rate identification number (RIN) as shown in Figure 1. RINs are assigned at the time rate information is first uploaded by the LSE through the MIDAS API. When an LSE uploads to an existing RIN, the correct RIN must be used at the time of upload. Figure 1 illustrates the six identifiers that comprise a RIN: Country, State, Distribution, Energy, Rate, and Location. The location portion of the RIN may consist of 1 to 10 characters depending on the specified location’s requirements.

Figure 1. Rate Identification Number Structure<br>
![Rate Identification Number Specification](img/RIN-structure.png)<br>
Source: California Energy Commission

Rate Identification Numbers do not change over time. The prices and values may change, but an electricity customer's RIN should not change unless their rate components or rate modifiers change or the utility or customer changes their rate tariff.

## Rate Information

To fulfill the requirements of the California's load management standards, MIDAS receives and shares information for all time-dependent rates for the three largest investor owned utilities (IOUs), the two largest publicly owned utilities (POUs) and the 14 largest CCAs in California. Time-dependent rates are rates that have prices which vary over the course of a day. Other California utilities and CCAs may use MIDAS to provide information and prices for their rates, but are not required to do so.

## SGIP GHG Emissions

MIDAS 2.0 provides greenhouse gas (GHG) emissions signals for all eleven WattTime SGIP grid regions in California through eleven region‑specific RINs following the pattern USCA‑SGIP‑MOER‑XXXX, where the final four characters identify the region. These RINs deliver continuous, predictable 5‑minute marginal operating emissions rates sourced from the [WattTime.org SGIP API](https://sgipsignal.com), which MIDAS retrieves, standardizes, and stores before serving through the API. Each RIN returns a full time series in grams of CO₂ per kilowatt-hour (g/kWh CO₂) and follows the same ValueData and HistoricalData structure as all other MIDAS 2.0 rate signals. For a list of the regions and region abbreviations please see WattTime’s SGIP webpage at: <https://sgipsignal.com/grid-regions>.

See [Appendix F](appendix-f.md) for the full GHG emissions RIN mapping and complete list of active RINs.

## CAISO Flex Alerts

Flex Alert information is provided through a single consolidated RIN, USCA‑FLEX‑ALRT‑0000, which returns a clean and predictable hourly signal indicating whether a California ISO Flex Alert is active. MIDAS periodically polls the CAISO Flex Alert webpage and stores the results in its database, supplying users with a binary hourly value of 1 when a Flex Alert is active and 0 when no alert is in effect, with no gaps in the time series. The RIN supports both standard MIDAS query types: realtime, which provides a 72‑hour window beginning at midnight Pacific on the request date, and alldata, which provides a 90‑day window ending at 23:59:59 Pacific on Day + 2. All timestamps are returned in UTC, and the data structure aligns with the format used for other MIDAS rate signals, ensuring consistency and ease of integration. 

For more information on the XML schema and uploads, see [Appendix A](appendix-a.md).

See [Appendix F](appendix-f.md) for the full FlexAlert RIN mapping and complete list of active RINs.

### Data Retention

| Tier | Window | Access Method |
|------|--------|---------------|
| Active | 2 years | MIDAS API (`realtime`, `alldata`, `historicaldata`) |
| Archive | Years 2–7 | Cold storage — contact midas@energy.ca.gov |
| Legacy | Older than 7 years | Request from CEC at midas@energy.ca.gov |

---

## 4. Authentication

### GET Requests

As of v2.0, **no token is required** for public GET requests to `GET /api/valuedata` with `SignalType`, `ID+QueryType`, or `LookupTable` parameters. The token endpoint remains available for LSE uploaders.

### POST Requests and LSE Endpoints

All upload and access-management endpoints require a **JWT Bearer token**.

**Step 1 — Register**

```
POST /api/registration
```

All fields are Base64-encoded. Required fields: `fullname`, `username`, `password`, `emailaddress`, `organization`.

On success: HTTP 200, verification email sent to provided address. You must verify the email as explained in Step 2.

**Step 2 — Verify email**

Once you get the confirmation code on your email, confirm programmatically:

```
POST /api/confirmemail?username=<username>&confirmation_code=<code>
```

**Step 3 — Get a token**

```
GET /api/token
Authorization: Basic <base64(username:password)>
```

The JWT token is returned in the `Token` response **header**. The response body contains metadata (`token_type`, `expires_in`). Tokens expire after **3,600 seconds**.

**Step 4 — Request upload access (LSE only)**

```
POST /api/uploadaccess/request
Authorization: Bearer <token>
Body: { "energy_code": "XX", "distribution_code": "XX" }
```

A CEC admin reviews and approves the request. You will receive an email confirmation when access is granted.

**Forgot password:**
```
POST /api/forgotpassword?username=<username>
POST /api/confirmpasswordreset   { "token": "...", "new_password": "..." }
```

**Resend verification email:**
```
POST /api/resendverification?username=<username>
```

---

## 5. GET Requests — Retrieving Data

### 5.1 Get RIN List

Returns all RINs of a given signal type.  
| Signal Type | Description |
|------|--------|
| 0 | ALL active RINs | 
| 1 | All active electricity rate RINs  | 
| 2 | All GHG EMission RINs | 
| 3 | Flex ALert RIN | 

```
GET /api/valuedata?SignalType={0|1|2|3}
```

**Response format (v2.0 — wrapped object):**

```json
{
  "Rates": [
    {
      "RateID": "USCA-BNBN-EVT2-0000",
      "SignalType": "Electricity Rates",
      "Description": "BEV-1 - CPP",
      "LastUpdated": "2026-05-22T14:56:34+0000"
    }
  ]
}
```

### 5.2 Get Rate Values

```
GET /api/valuedata?ID={RIN}&QueryType={realtime|alldata}
```

| QueryType | Window |
|-----------|--------|
| `realtime` | 72 hours from midnight Pacific on the request date |
| `alldata` | 90 days ending at 23:59:59 Pacific on Day+2 |

Query parameter names are **case-insensitive**. The DateStart, DateEnd, TimeStart, and TimeEnd will be returned in UTC.

Please note: If you submit a realtime query on July 1, 2026, the data window begins at 12:00:00 AM Pacific Time on that same day. However, because MIDAS returns all timestamps in UTC, the first data point will appear with a DateStart and TimeStart of 07:00:00 AM on July 1, 2026, which corresponds to midnight Pacific Time.  

**Example — Flex Alert realtime:**
```
GET /api/valuedata?ID=USCA-FLEX-ALRT-0000&QueryType=realtime
```

**Example — GHG emissions for SMUD region:**
```
GET /api/valuedata?ID=USCA-SGIP-MOER-SMUD&QueryType=alldata
```

### 5.3 Get Historical Data

For date-range queries beyond the `alldata` 90-day window (up to 6 months per request):

```
GET /api/historicaldata/{rate_id}?startdate=YYYY-MM-DD&enddate=YYYY-MM-DD
```

No authentication required. Maximum range: 6 months per request. Get data as old as 2 years from the date of query.

### 5.4 Get Lookup Tables

```
GET /api/valuedata?LookupTable={TableName}
```

Available tables: `Distribution`, `Energy`, `StateProvince`, `Unit`, `RateType`, `SignalType`, `DayType`. Note: `Holiday` and `TimeZone` tables have been removed in v2.0.

---

## 6. POST Requests — Uploading Data (LSE Users)

### Prerequisites

1. Registered and email-verified account
2. Upload access approved by CEC for the relevant distribution and energy codes
3. Valid Bearer token (obtained via `GET /api/token`)

### Upload Rate Data

```
POST /api/valuedata
Authorization: Bearer <token>
Content-Type: application/json
Body: <ValueData JSON or XML payload>
```

On success, the API returns **HTTP 202 Accepted** (not 200) with a `jobID`:

```json
{
  "message": "Rate data upload accepted and queued for processing",
  "jobID": "019e4c2d-6906-7cc4-a447-8259d107be54",
  "status": "PROCESSING",
  "statusUrl": "https://midasapi.energy.ca.gov/api/jobs/019e4c2d-6906-7cc4-a447-8259d107be54",
  "createdDateTime": "2026-06-22T21:18:03Z"
}
```

**Store the `jobID`.** Use it to poll validation status (see Section 8).

### Upload Rules

- Accepted intervals: `1 hour`, `15 minutes`, `5 minutes` only.
- Interval assigned on first upload for a RIN — cannot change.
- All units in a single upload must have the same number of data points and the same interval.
- Missing values within a unit must be filled with zeros to maintain alignment.
- Uploading past-dated data is accepted but triggers a warning notification.
- A gap between the latest data in MIDAS and the earliest timestamp in your upload triggers a gap warning.

---

## 7. Python Examples

Refer to [Appendix G](appendix-g.md) for Python example codes.

---

## 8. Jobs Endpoint — Tracking Upload Status

After every `POST /api/valuedata`, you receive a `jobID`. Poll the jobs endpoint to get the final validation result.

### How It Works

```
POST /api/valuedata  →  HTTP 202 + jobID
         ↓
MIDAS backend runs full validation asynchronously
         ↓
GET /api/jobs/{jobID}  →  PROCESSING | COMPLETE | VALIDATION_FAILED
```

Job retention supports a configurable TTL (currently retained for 7 days), then automatically deleted. 

### Get a Single Job

```
GET /api/jobs/{jobID}
Authorization: Bearer <token>
```

**Status values:**

| Status | Meaning |
|--------|---------|
| `PROCESSING` | Validation is still running |
| `COMPLETE` | Upload accepted and processed successfully |
| `VALIDATION_FAILED` | Upload rejected — see `validationIssues` |

**Example response — successful upload:**

```json
{
  "id": "019e4c2d-6906-7cc4-a447-8259d107be54",
  "createdDateTime": "2026-06-22T21:18:03Z",
  "modificationDateTime": "2026-06-22T21:18:45Z",
  "status": "COMPLETE",
  "username": "your_username",
  "uploadFormat": "JSON",
  "summary": {
    "rins": [
      {
        "rin": "USCA-TSTS-0354-TEST",
        "units": ["KWH", "KW"],
        "numTimePeriods": 96,
        "firstDateTime": "2026-06-23T07:00:00Z",
        "lastDateTime": "2026-06-27T07:00:00Z"
      }
    ],
    "processingTimeMs": 2530
  }
}
```

**Example response — validation failure:**

```json
{
  "id": "019e4c7f-034a-7a8e-a2f3-997135585ff9",
  "status": "VALIDATION_FAILED",
  "summary": {
    "rins": [
      {
        "rin": "USCA-TSTS-0354-TEST",
        "validationIssues": [
          {
            "type": "ERROR",
            "code": "UNEQUAL_INTERVAL",
            "message": "There are more than one interval length in the data. All interval lengths must be identical, and one of 5, 15, or 60 minutes."
          },
          {
            "type": "WARNING",
            "code": "MISSING_INTERVAL",
            "message": "There are one or more missing intervals in the upload."
          }
        ]
      }
    ]
  }
}
```

### List Recent Jobs

```
GET /api/jobs
Authorization: Bearer <token>
```

Optional query parameter: `detailed=true` to include per-RIN summaries in the list response. Results are returned newest-first.

### JobID Format

Job IDs are **UUID version 7** — time-sortable, globally unique identifiers. Format: `019e4c2d-6906-7cc4-a447-8259d107be54`.

---

## 9. System Health Endpoints

These endpoints require no authentication and are intended for monitoring.

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Basic API health check — returns `status`, `service`, `version` |
| `GET /watttime/status` | Simple up/down for WattTime token service |
| `GET /watttime/health` | Full WattTime health: token expiry, circuit breaker, cache |
| `GET /watttime/health/comprehensive` | Health score 0–100, all metrics |
| `GET /watttime/metrics` | Token refresh rates, latency, circuit breaker activations |
| `GET /caiso/status` | Simple up/down for CAISO Flex Alert service |
| `GET /caiso/health` | Full CAISO health: circuit breaker, cache, metrics |

---

## 10. Error Handling

### Standard HTTP Status Codes

| Code | Meaning | Common Cause |
|------|---------|--------------|
| `200 OK` | Success | |
| `202 Accepted` | Upload received and queued | `POST /api/valuedata` |
| `400 Bad Request` | Invalid parameters or Validation Error | Bad RIN, invalid QueryType, Data Invalid |
| `401 Unauthorized` | Missing or invalid token | Token expired or not provided |
| `403 Forbidden` | Insufficient permissions | Unauthorized energy code 'MC'. User is authorized for ['PG', 'TS'] |
| `404 Not Found` | RIN not found | RIN doesn't exist in MIDAS |
| `422 Unprocessable Entity` | Validation error | Missing required fields |
| `503 Service Unavailable` | Upstream dependency down | WattTime or CAISO unavailable |


### Upload Validation Issue Codes

| Code | Type | Description |
|------|------|-------------|
| `UNEQUAL_INTERVAL` | ERROR | Multiple interval lengths in one upload |
| `MISSING_INTERVAL` | WARNING | One or more time intervals absent |
| `INVALID_RIN` | ERROR | RIN not recognized or not authorized |
| `INTERVAL_MISMATCH` | ERROR | Interval differs from RIN's assigned interval |
| `DATA_GAP` | WARNING | Gap between latest MIDAS data and upload start |
| `PAST_DATA` | WARNING | Upload contains timestamps in the past |

---

## 11. Rate Limits

MIDAS is protected by CEC firewall rules and request throttling. Exceeding the rate limit returns HTTP `429 Too Many Requests`. If you receive `429` errors regularly, contact midas@energy.ca.gov to discuss your use case.

---

# Appendices

## Appendix A Uploading to MIDAS

[Appendix A](appendix-a.md) discusses rate and holiday upload and links to example upload documents.

## Appendix B Acronyms and Glossary

[Appendix B](appendix-b.md) contains a list of acronyms and a glossary.

## Appendix C Lookup Tables

[Appendix C](appendix-c.md) contains a list of all lookup tables available through MIDAS.

## Appendix D MIDAS Architecture

[Appendix D](appendix-d.md) contains a diagram of the MIDAS API service architecture.

## Appendix E MIDAS Data Dictionary

[Appendix E](appendix-e.md) contains the description of the data dictionary and a link to download it.

## Appendix F SGIP and FlexAlert RIN mapping in MIDAS

[Appendix F](appendix-f.md) contains the WattTime's SGIP and CAISO's FlexAlert RIN mapping in MIDAS

## Appendix G Python Code Examples

[Appendix G](appendix-g.md) contains example codes in MIDAS to upload data to MIDAS and access MIDAS data
