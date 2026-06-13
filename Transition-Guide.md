# MIDAS v1.0 → v2.0 Transition Guide

**Applies to:** All existing MIDAS users migrating from v1.0 to v2.0 

**Release Date:** June 22, 2026 

**Contact:** midas@energy.ca.gov 

---

## Before You Begin

This guide walks through every change that requires action before June 22, 2026. Read the section that applies to your role — **GET users (data consumers)** or **POST users (LSE uploaders)** — and work through each checklist item.

Not sure what role applies to you? If you only query MIDAS data, you are a GET user. If you submit rate data on behalf of a utility or CCA, you are a POST user. Some teams are both.

---

## Table of Contents

1. [Changes for GET Users (Data Consumers)](#part-1--changes-for-get-users-data-consumers)
   - [Step 1.1 — Token is now optional for GET requests](#step-11--token-is-now-optional-for-get-requests)
   - [Step 1.2 — Update GHG RINs](#step-12--update-ghg-rins)
   - [Step 1.3 — Update GHG unit handling (kg → g)](#step-13--update-ghg-unit-handling-kg--g)
   - [Step 1.4 — Update Flex Alert RINs](#step-14--update-flex-alert-rins)
   - [Step 1.5 — Update RINList response parsing](#step-15--update-rinlist-response-parsing)
   - [Step 1.6 — Query window changes](#step-16--query-window-changes)
   - [Step 1.7 — Removed endpoints](#step-17--removed-endpoints)
     
2. [Changes for POST Users (LSE Uploaders)](#part-2--changes-for-post-users-lse-uploaders)
   - [Step 2.1 — Re-register](#step-21--re-register)
   - [Step 2.2 — Request upload access](#step-22--request-upload-access)
   - [Step 2.3 — Handle the new upload response](#step-23--handle-the-new-upload-response)
   - [Step 2.4 — Poll the Jobs endpoint instead of waiting for email](#step-24--poll-the-jobs-endpoint-instead-of-waiting-for-email)
   - [Step 2.5 — Update token expiry logic](#step-25--update-token-expiry-logic)
   - [Step 2.6 — Validate your upload interval consistency](#step-26--validate-your-upload-interval-consistency)
  
---

## Part 1 — Changes for GET Users (Data Consumers)

### Checklist

- [ ] Remove token calls or authorization for public GET requests (no registration or account needed to access MIDAS data)
- [ ] Update the `USCA-SGIP-SGRT/SGFC/SGHT` RINs to `USCA-SGIP-MOER-{REGION}`
- [ ] Update `USCA-FLEX-FXRT/FXFC/FXHT` RINs to `USCA-FLEX-ALRT-0000`
- [ ] Multiply GHG thresholds, stored values, and chart axes by 1,000 (kg → g)
- [ ] Update any code that parses the RINList response (now a keyed object)
- [ ] Update `alldata` integrations if you depend on specific window size (now 90 days)
- [ ] Remove any calls to `HistoricalRINList`, `Holiday`, or `TimeZone` endpoints/lookup tables

---

### Step 1.1 — Token is now optional for GET requests

In v2.0, the following endpoints are **publicly accessible without a token or authorization**:

```
GET /api/valuedata?SignalType=N
GET /api/valuedata?ID={RIN}&QueryType={realtime|alldata}
GET /api/historicaldata/{rate_id}
```

You may remove the `Authorization: Bearer` header from those calls. Existing code that sends a valid token will continue to work — no forced change here, just simplification available.

---

### Step 1.2 — Update GHG RINs

All `SGRT`, `SGFC`, and `SGHT` RINs are gone. Calling them returns `410 Gone`. Replace each with the corresponding `MOER` RIN.

**Mapping — replace old RIN + call pattern with new:**

| Old RIN (v1.0) | v2.0 Replacement |
|---|---|
| `USCA-SGIP-SGRT-PACW` | `GET /api/valuedata?ID=USCA-SGIP-MOER-PACW&QueryType=realtime` |
| `USCA-SGIP-SGFC-PACW` | `GET /api/valuedata?ID=USCA-SGIP-MOER-PACW&QueryType=realtime` |
| `USCA-SGIP-SGHT-PACW` | `GET /api/valuedata?ID=USCA-SGIP-MOER-PACW&QueryType=alldata` |
| `USCA-SGIP-SGRT-SDGE` | `USCA-SGIP-MOER-SDGE` + `realtime` |
| `USCA-SGIP-SGFC-SDGE` | `USCA-SGIP-MOER-SDGE` + `realtime` |
| `USCA-SGIP-SGHT-SDGE` | `USCA-SGIP-MOER-SDGE` + `alldata` |
| `USCA-SGIP-SGRT-PGE` | `USCA-SGIP-MOER-PGE` + `realtime` |
| `USCA-SGIP-SGFC-PGE` | `USCA-SGIP-MOER-PGE` + `realtime` |
| `USCA-SGIP-SGHT-PGE` | `USCA-SGIP-MOER-PGE` + `alldata` |
| *(repeat for SCE, SMUD, LADWP, TID, IID, NVENERGY, WALC, P2)* | *(same pattern)* |

Full mapping table in [Appendix A](appendix-a.md).

---

### Step 1.3 — Update GHG unit handling (kg → g)

This is the highest-risk change for data consumers. GHG values are now **1,000× larger** than before.

```python
# v1.0 — values were in kg/kWh CO2
# threshold_kg = 0.300  # 300 g/kWh in old units

# v2.0 — values are in g/kWh CO2
threshold_g = 300.0  # same physical value, correct new units

# If you are comparing against a stored historical threshold from v1.0:
old_threshold_kg = 0.300
new_threshold_g = old_threshold_kg * 1000  # = 300.0
```

**Where to check:**
- Comparison thresholds and trigger logic
- Chart Y-axis scales and labels
- Stored values in databases seeded from v1.0 data
- Unit labels shown to end users
- Alert configurations

---

### Step 1.4 — Update Flex Alert RINs

| Old RIN (v1.0) | v2.0 Replacement |
|---|---|
| `USCA-FLEX-FXRT-0000` | `USCA-FLEX-ALRT-0000` + `QueryType=realtime` |
| `USCA-FLEX-FXFC-0000` | `USCA-FLEX-ALRT-0000` + `QueryType=realtime` |
| `USCA-FLEX-FXHT-0000` | `USCA-FLEX-ALRT-0000` + `QueryType=alldata` |

The new RIN returns a clean hourly time series: `Value=1` (Flex Alert active), `Value=0` (not active). No gap-filling needed — every hour in the window is present.

**Before (v1.0) — three calls:**
```python
# Had to call FXRT + FXFC separately and combine manually
rt = requests.get(url, params={"ID": "USCA-FLEX-FXRT-0000", "QueryType": "realtime"})
fc = requests.get(url, params={"ID": "USCA-FLEX-FXFC-0000", "QueryType": "realtime"})
```

**After (v2.0) — one call:**
```python
response = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={"ID": "USCA-FLEX-ALRT-0000", "QueryType": "realtime"}
)
```

---

### Step 1.5 — Update RINList response parsing

The response shape changed from a bare array to a keyed object.

```python
# v1.0 — response was a list
data = response.json()  # [{...}, {...}]
for rin in data:
    print(rin["RateID"])

# v2.0 — response is a keyed object
data = response.json()  # {"Rates": [{...}, {...}]}
for rin in data.get("Rates", []):
    print(rin["RateID"])
```

The key name reflects the signal type: `"Rates"` for electricity rates, `"GHGEmissions"` for GHG signals, `"FlexAlerts"` for Flex Alert signals, and `"All"` when `SignalType=0`.

---

### Step 1.6 — Query window changes

| Query | v1.0 | v2.0 |
|-------|------|------|
| `realtime` | Single current value (GHG) or pass-through (Flex Alert) | 72-hour window from midnight Pacific on request date |
| `alldata` | No defined cap | 90-day window ending at 23:59:59 Pacific on Day+2 |

If your application depends on a specific window size, update your downstream logic. For data older than 90 days, use `GET /api/historicaldata/{rate_id}` with explicit `startdate`/`enddate` parameters. The historicaldata query is limited to request 6 months of data at a time.

---

### Step 1.7 — Removed endpoints

Remove any calls to the following — they no longer exist in v2.0:

- `GET /api/historicallist` (HistoricalRINList) — no replacement
- - `GET /api/Holiday` (Holiday) — no replacement
- `GET /api/valuedata?LookupTable=Holiday` — no replacement
- `GET /api/valuedata?LookupTable=TimeZone` — no replacement

---

## Part 2 — Changes for POST Users (LSE Uploaders)

### Checklist

- [ ] Re-register account on the new system
- [ ] Verify email address
- [ ] Submit upload access request via API
- [ ] Wait for CEC approval email
- [ ] Update upload code to handle HTTP 202 (not 200) on success
- [ ] Store `jobID` from upload response
- [ ] Implement job polling logic (`GET /api/jobs/{jobID}`)
- [ ] Update token refresh logic (now 3,600s expiry)
- [ ] Verify interval consistency and all validations in upload payloads

---

### Step 2.1 — Re-register

Your v1.0 credentials will not work for uploads on the v2.0 system. You must re-register.

```
POST /api/registration
```

All fields Base64-encoded: `fullname`, `username`, `password`, `emailaddress`, `organization`.

After submitting, check your email and click the verification link. If the email doesn't arrive within a few minutes, use:

```
POST /api/resendverification?username=<your_username>
```

---

### Step 2.2 — Request upload access

After verifying your email and obtaining a token, submit an upload access request for each distribution/energy code combination you need:

```
POST /api/uploadaccess/request
Authorization: Bearer <token>
Body: { "energy_code": "XX", "distribution_code": "XX" }
```

You will receive an email when CEC approves your request. **You cannot upload until access is approved.** Contact midas@energy.ca.gov if you do not receive approval within 2 business days.

---

### Step 2.3 — Handle the new upload response

The POST /api/valuedata response changed in two ways:

1. HTTP status is now **202 Accepted** (was 200 OK).
2. Response body now contains `jobID` and `statusUrl` (the old `sqsMessageId` is removed).

```python
# v1.0
response = requests.post(url, json=payload, headers=headers)
if response.status_code == 200:
    print("Upload accepted")

# v2.0
response = requests.post(url, json=payload, headers=headers)
if response.status_code == 202:
    result = response.json()
    job_id = result["jobID"]       # store this
    status_url = result["statusUrl"]  # for polling
    print(f"Accepted. Job: {job_id}")
```

---

### Step 2.4 — Poll the Jobs endpoint instead of waiting for email

The v2.0 upload workflow is fully programmatic. After receiving a `jobID`, poll `GET /api/jobs/{jobID}` to check validation status.

```python
import time
import requests

BASE_URL = "https://midasapi.energy.ca.gov"

def wait_for_job(token: str, job_id: str, timeout_seconds: int = 120) -> dict:
    headers = {"Authorization": f"Bearer {token}"}
    deadline = time.time() + timeout_seconds
    
    while time.time() < deadline:
        time.sleep(5)
        resp = requests.get(f"{BASE_URL}/api/jobs/{job_id}", headers=headers)
        resp.raise_for_status()
        job = resp.json()
        
        if job["status"] in ("COMPLETE", "VALIDATION_FAILED"):
            return job
    
    raise TimeoutError(f"Job {job_id} timed out after {timeout_seconds}s")

job = wait_for_job(token, job_id)

if job["status"] == "COMPLETE":
    print("Upload complete!")
else:
    for rin in job["summary"]["rins"]:
        for issue in rin.get("validationIssues", []):
            print(f"{issue['type']} [{issue['code']}]: {issue['message']}")
```

**Important:** Jobs are retained for 1 week. If you do not poll within that window, the job record is deleted. Store the `jobID` persistently if you need it for reconciliation.

---

### Step 2.5 — Update token expiry logic

Tokens now expire after **3,600 seconds** (1 hour). If your code refreshed the token every 9 minutes to stay within the old 600-second window, you can now safely extend that to ~55–58 minutes.

```python
import time

TOKEN_EXPIRY_SECONDS = 3600
REFRESH_BUFFER_SECONDS = 120  # refresh 2 minutes before expiry

token = None
token_acquired_at = 0

def get_valid_token():
    global token, token_acquired_at
    if token is None or (time.time() - token_acquired_at) > (TOKEN_EXPIRY_SECONDS - REFRESH_BUFFER_SECONDS):
        token = get_token("your_username", "your_password")
        token_acquired_at = time.time()
    return token
```

---

### Step 2.6 — Validate your upload interval consistency

Before uploading, ensure:

1. All intervals in your payload are one of: **5 minutes**, **15 minutes**, or **1 hour**.
2. All units in the upload have the **same number of data points**.
3. If a unit has fewer real values than required, fill missing slots with **zeros**.

```python
def validate_upload_consistency(rate_data: list) -> bool:
    """Check that all units have the same number of time periods."""
    counts = [len(unit["values"]) for unit in rate_data]
    if len(set(counts)) > 1:
        print(f"Unit count mismatch: {counts}")
        return False
    return True
```

---

## Part 3 — Base URL

All calls must go to:

```
https://midasapi.energy.ca.gov
```

Update any hardcoded URLs or environment variables in your deployment configuration.

---

## Timeline Summary

| Date | Event |
|------|-------|
| Before June 22, 2026 | Update GET integrations (RINs, unit handling, response parsing) |
| **On or Before June 22, 2026** | **Complete re-registration and upload access request** |
| **June 22, 2026** | **MIDAS v2.0 goes live.** |

---

## Getting Help

If you encounter issues during migration:

- Email: **midas@energy.ca.gov**
- Include your username, the endpoint you are calling, the full request (redact your password/token), and the response you received.

---

*California Energy Commission | MIDAS v2.0 Transition Guide | June 22, 2026*
