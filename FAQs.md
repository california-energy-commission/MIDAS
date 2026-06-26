# MIDAS v2.0 — Frequently Asked Questions

**Last updated:** June 22, 2026 | **Contact:** <midas@energy.ca.gov>

## General

**Q: What is MIDAS v2.0?**

MIDAS v2.0 is a major upgrade to the California Energy Commission's Market Informed Demand Automation Server, released on June 22, 2026. It delivers improved infrastructure, simplified data access, consolidated GHG and Flex Alert signals, a new programmatic upload tracking system (the Jobs endpoint), and an OpenADR 3.1-compatible endpoint.

**Q: What is the base URL for the production API?**

`https://midasapi.energy.ca.gov`

**Q: Where can I find the full API specification?**

The interactive OpenAPI documentation is available at `https://midasapi.energy.ca.gov/docs`. The raw OpenAPI JSON is at `https://midasapi.energy.ca.gov/openapi`.

**Q: I have a question not covered here. How do I get help?**

Email the MIDAS team at **<midas@energy.ca.gov>**. Include your username, the endpoint you are calling, your request (redact your password and token), and the full response you received.

## GET / Data Access

**Q: Do I still need to register to query MIDAS data?**

No. As of v2.0, all public GET endpoints are accessible without a token or registration. You only need to register if you intend to upload data as a Load Serving Entity/Utility.

**Q: I called an old GHG RIN and got an error. What happened?**

The 33 legacy SGIP GHG RINs (`USCA-SGIP-SGRT-{REGION}`, `USCA-SGIP-SGFC-{REGION}`, `USCA-SGIP-SGHT-{REGION}`) were consolidated into 11 new MOER RINs in v2.0. Use `USCA-SGIP-MOER-{REGION}` with `QueryType=realtime` or `QueryType=alldata`.

**Q: I called the Flex Alert RIN and got an error.**

The three v1.0 Flex Alert RINs (`USCA-FLEX-FXRT-0000`, `USCA-FLEX-FXFC-0000`, `USCA-FLEX-FXHT-0000`) are replaced by `USCA-FLEX-ALRT-0000`. Use `QueryType=realtime` for current and upcoming status, or `QueryType=alldata` for the past 90 days.

**Q: My GHG values suddenly jumped 1,000×. Is the API broken?**

No — this is expected. The unit of measure for GHG emissions changed from `kg/kWh CO2` to `g/kWh CO2`. A value that was `0.333` in v1.0 is `333.0` in v2.0. Both represent the same physical quantity. Update your thresholds, chart scales, and stored comparisons by multiplying v1.0 values by 1,000.

**Q: How much data does `QueryType=alldata` return now?**

The `alldata` window is now a fixed **90 days** ending at 23:59:59 Pacific on Day+2. Previously there was no defined cap. For data older than 90 days, use `GET /api/historicaldata/{rate_id}?startdate=YYYY-MM-DD&enddate=YYYY-MM-DD` (maximum 1 year per request).

**Q: How much data does `QueryType=realtime` return?**

A 72-hour window starting at 00:00:00 Pacific on the day of the request. For GHG signals, this is 5-minute intervals (approximately 864 entries). For Flex Alert, this is hourly (72 entries).

**Q: The RINList response format changed and now my JSON parsing is broken.**

In v2.0, the `GET /api/valuedata?SignalType=N` response is a keyed object, not a bare array. For electricity rates, the key is `"Rates"`. Update your parsing:

```python
data = resp.json()
rin_list = data.get("Rates", [])  # or "GHGEmissions", "FlexAlerts", "All"
```

**Q: Where did the `HistoricalRINList` endpoint go?**

It was removed in v2.0 with no direct replacement. To get the full list of available RINs, use `GET /api/valuedata?SignalType=0` (returns all signal types).

**Q: Where did the `Holiday` and `TimeZone` lookup tables go?**

Both were removed in v2.0 with no replacement.

**Q: How do I get data older than 2 years?**

Data from 2 to 7 years old is stored in a cold storage archive and is not directly accessible via the API. MIDAS team is working to make this data available and will release it in an upcoming upgrade. Contact the MIDAS team at <midas@energy.ca.gov> for any questions. Data older than 7 years can also be requested from CEC directly.

**Q: My token was valid for 10 minutes before. Why does it say `expires_in: 3600` now?**

Token lifetime has been extended to 3,600 seconds (1 hour) in v2.0. This is correct. Update your token refresh logic if you were refreshing every 9 minutes.

## POST / Uploading Data (LSE Users)

**Q: Do I need to re-register for v2.0?**

Yes, all existing LSE upload users must re-register. The identity system has been migrated and v1.0 credentials do not carry over. Register at `POST /api/registration`, verify your email, and submit an upload access request via `POST /api/uploadaccess/request`.

**Q: My upload returned HTTP 202 instead of 200. Is that an error?**

No. `HTTP 202 Accepted` is the correct success response for uploads in v2.0. It means your file passed initial validation and has been queued for processing. The `jobID` in the response body is your tracking identifier for final validation results. You can check the status of your upload using the new `jobs` endpoint.

**Q: How long should I wait before polling the job status?**

Most uploads complete validation within 10–30 seconds. A reasonable polling strategy is to wait 5 seconds, then check every 5 seconds thereafter, up to a 2-minute timeout. The status will be `PROCESSING` until validation completes.

**Q: My job is stuck in `PROCESSING` for a long time. What should I do?**

If a job has not reached `COMPLETE` or `VALIDATION_FAILED` within 5 minutes, contact <midas@energy.ca.gov> with your `jobID` and the time you submitted the upload.

**Q: What does `VALIDATION_FAILED` mean? Is my data lost?**

`VALIDATION_FAILED` means the upload did not pass MIDAS's data quality and regulatory validation checks. The uploaded file was not persisted to the MIDAS database. Check the `validationIssues` array in the job response — each issue has a `code` and a human-readable `message` explaining what needs to be fixed. Correct the issues and re-submit.

**Q: What is the `UNEQUAL_INTERVAL` error?**

Your upload contains timestamps that are not evenly spaced, or contains more than one interval length. MIDAS requires all records in an upload to use a single consistent interval of either 5 minutes, 15 minutes, or 60 minutes. Check your data for any gaps, duplicates, or mixed intervals.

**Q: What is the `INTERVAL_MISMATCH` error?**

The interval in your upload does not match the interval that was assigned to this RIN on its first upload. For example, if the RIN was first uploaded with hourly data, all subsequent uploads must also be hourly. The assigned interval cannot be changed.

**Q: Can I upload XML or does it have to be JSON?**

Both are accepted. The API converts XML to JSON internally. The `uploadFormat` field in the job response will reflect whether you submitted XML or JSON.

**Q: My upload has fewer values for one unit than the others. What should I do?**

Fill missing entries with appropriate values to ensure all units in the upload have the same number of data points. MIDAS requires cross-unit consistency within a single upload payload.

**Q: Can my account now have multiple distribution or energy codes?**

Yes. A single account can hold upload authorization for multiple distribution codes and multiple energy codes. Submit a separate `POST /api/uploadaccess/request` for each additional code pair. Each request requires CEC approval.

**Q: How long does upload access approval take?**

CEC reviews requests within 2 business days. If you do not receive confirmation within that window, email <midas@energy.ca.gov>.

**Q: I uploaded past-dated data and got a warning. Was my upload rejected?**

No. The `PAST_DATA` warning is informational. Your upload was accepted. This warning alerts both you and the MIDAS team that the submission includes timestamps in the past.

**Q: I got a `DATA_GAP` warning. Does that mean my upload failed?**

No. The `DATA_GAP` warning is also informational. It indicates that there is a gap between the most recent data MIDAS has for this RIN and the earliest timestamp in your upload. Your upload was accepted.

**Q: How long are job records retained?**

24 hours from the time of upload creation, after which records are automatically deleted from DynamoDB using TTL. Store the `jobID` persistently if you need it for audit or reconciliation purposes.

## System and Reliability

**Q: How do I check if the MIDAS API is operational?**

Use `GET /health` for a basic status check. For WattTime and CAISO dependency status, use `GET /watttime/status` and `GET /caiso/status`. These endpoints require no authentication and return a simple `status: healthy/unhealthy` response.

**Q: What are the new WattTime and CAISO health endpoints for?**

They expose operational metrics about MIDAS's upstream dependencies — the WattTime SGIP API (for GHG data) and the CAISO Flex Alert service. They are intended for monitoring integrations, not for end-user applications. If WattTime or CAISO is unreachable, MIDAS will serve cached values when available and return `503 Service Unavailable` otherwise.

---

*California Energy Commission | MIDAS v2.0 FAQ | June 22, 2026*
