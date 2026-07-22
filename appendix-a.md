# Appendix A - Uploading to MIDAS

Only CEC approved entities can upload to MIDAS. After establishing a MIDAS account by registering through the `registration` API endpoint and verifying your email, request upload capabilities through the `uploadaccess/request` API endpoint with your LSE account username. If you do not receive a confirmation email for the granted access, please reach out by email at <midas@energy.ca.gov>. CEC staff will review your request and respond.

Note: Utilities may be assigned distribution and energy company access to MIDAS. These assignments change what rates the user is allowed to upload.

## Rate Upload Support

LSEs are encouraged to reach out to CEC staff at <midas@energy.ca.gov> with any questions or issues with MIDAS. Staff will do their best to answer any questions and address issues quickly.

The rate upload XML schema is available through the MIDAS API. The version from the API is the canonical one. However, we are also providing the XML schema document as part of this documentation. [XML Schema Document](support-docs/MIDAS-upload-XML-schema.xsd).

## Rate Definitions

MIDAS is designed to provide energy users with the electricity price information they need to decide when to use energy. While it would be useful for electricity users to be able to use the data from MIDAS to try to estimate electricity bill totals, some current billing structures such as tiered rates make this impossible without private customer-specific data. Electricity users need information (prices) on the volumetric portions of their rates to effectively optimize energy use. This tells them how much each kWh costs during a particular time period. Energy users can use this information to decide when to use electricity. They can also set their controllable devices to adjust when they use electricity based on the same information. When enough users shift their electricity use to cheaper and greener hours when more renewable energy is available, this can reduce electricity costs for all users of the grid.

## Upload Rates

Rates must be updated in MIDAS before prices change so that electricity users can plan for and act on price changes. LSEs can upload rates to MIDAS in either XML or JSON format. When an LSE uploads a new set of values for a rate, the previous set are removed from the Value table, but retained in the ValueHistory table.

### Rate Examples

Here are some example rate upload documents to help LSEs format rate uploads. Each example XML or JSON file only contains a few days worth of data (March 1-3, 2023) for readability.

* **TOU rate** An example of a TOU rate with hourly values. A full year of this rate would have 8760 ValueData blocks<br>
[XML TOU example](support-docs/MIDAS_Test_TOU.xml)<br>
[JSON TOU example](support-docs/MIDAS_Test_TOU.json)
* **TOU rate with time-varying demand charges** This file contains the same rate as the TOU file, but also contains ValueData blocks for time-dependent demand charges<br>
[XML TOU+demand example](support-docs/MIDAS_Test_Demand.xml)<br>
[JSON TOU+demand example](support-docs/MIDAS_Test_Demand.json)

### Assigning a RIN

The first step in uploading a rate to MIDAS is to determine the Rate Identification Number (RIN). See [Figure 1](README.md#database-structure) for the RIN structure. If a rate already exists in MIDAS and you don't have the RIN available, refer to the [Get RIN LIST](README.md#get-rin-list) section in the main documentation to download the list of RINs, and make sure to use the existing RIN. If the rate has never been previously uploaded to MIDAS, you will need to determine the RIN before uploading.

For California-based LSEs, the first four characters of the RIN will always be **USCA**, showing that the LSE is in the United States (US) and California (CA). For utility users in other countries and states/provinces, refer to the **Country** and **State** lookup tables to find your country and state code.

The next step is to determine the distribution and energy company codes that make up the next four characters of the RIN. Refer to the **Distribution** and **Energy** lookup tables to find the two character codes that correspond to your distribution and energy companies. There are two pathways for this determination. The first applies to all rates for large POUs and to rates with all-in prices for large IOUs and CCAs. The second applies to unbundled rates from large IOUs and CCAs that are not compiled into a single price.

1. *For rates that contain both distribution and energy costs in the price:* Use the distribution code from the distribution provider and the energy code from the energy provider. For example, Marin Clean Energy would use the Pacific Gas and Electric distribution company code **PG** and the Marin Clean Energy energy code **MC**, yielding **PGMC** for the second four characters.

2. *For partial rates from large IOUs and large CCAs that contain only costs from either distribution or energy:*

    1. IOUs uploading their unbundled delivery-only rates, use the distribution code from the distribution provider and XX in place of the energy provider. For example, Pacific Gas and Electric uses the PG&E distribution company code **PG** and **XX** in place of the energy code, yielding **PGXX** for the second four characters.

    2. CCAss uploading their unbundled energy-only rates, use the energy code from the energy provider and XX in place of the distribution provider. For example, Marin Clean Energy uses **XX** in place of the  distribution company code and the Marin Clean Energy energy code **MC**, yielding **XXMC** for the second four characters.

The third set of four characters are open for the LSE to define. These four alphanumeric characters (containing only uppercase English letters and the numeric digits 0-9) should, to the extent possible, reflect the rate. For example, a commercial TOU rate could have the four characters **CTOU**, or a critical-peak rate could have the characters **CPP2**.

The final four to 10 characters are the location code. For rates with no specific location, use the four character code **0000**. The list of allowable location codes are available in the **Location** lookup table. If your LSE has location codes to add to that table, contact the MIDAS team at <midas@energy.ca.gov> to request the addition of those codes.

Putting this together for a full example, the full RIN for an all-in, "bundled", rate at PG&E could be **USCA-PGPG-CTOU-0000**.

### Rate Upload Data Structure

The canonical requirements for the rate upload structure is the XML rate upload schema available through the API at the `/ValueData` endpoint. You can use this schema verify that your rate upload is structured correctly prior to uploading to MIDAS using XML validation software.

**Endpoint:** `/ValueData`

**HTTP Request:** `GET https://midasapi.energy.ca.gov/api/ValueData`

**Authorization:** Bearer

**Query Parameters:** None

**Body Parameters:** None

#### Rate Upload Python Example

```python
import requests
import xml.etree.ElementTree as ET

headers = {'accept': 'application/json', 'Authorization': "Bearer " + token}
url = 'https://midasapi.energy.ca.gov/api/ValueData'
pricing_response = requests.get(url, headers = headers)

element = ET.XML(json.loads(pricing_response.text))
ET.indent(element)
print(ET.tostring(element, encoding='unicode'))
```

#### Streaming Rate Structure

All rates uploaded to MIDAS should be in a "streaming" structure. This is a time-series structure where every hour (or sub-hourly period) has an entry. There needs to be at least one value for every hour, if the rate changes with a frequency higher than hourly, it needs to have one entry for each period where the rate could change, even when it does not change.

If the rate has more than one value in each interval each value will have the fields shown in the next paragraph. For example, a rate with asymmetric prices where the import price is different from the export price will need to have the following fields for each hour for the import price using the unit "\$/kWh" and also the following fields for the export price with the unit "export \$/kWh". When uploading export prices, use positive values for the amount the energy user will be paid for exporting electricity.

Each time period (interval) contains these fields:

- **DateStart** *required* This is the date in UTC when the rate interval starts.
- **TimeStart** *required* This is the time in UTC when the rate interval starts.
- **DateEnd** *required* This is the date in UTC when the rate interval ends.
- **TimeEnd** *required* This is the time in UTC when the rate interval ends.
- **DayStart** *required* Indicates the day type (1–8) in local time when the rate interval begins. Valid day types are listed in the DayType lookup table.
- **DayEnd** *required* Indicates the day type (1–8) in local time when the rate interval ends. Valid day types are listed in the DayType lookup table.
- **Value** *required* This is the value (usually the price) that applies to the interval.
- **Unit** *required* This is the unit that applies to the Value (usually $/kWh) that applies to the interval. Allowable units are in the Unit lookup table.
- **ValueName** *required* A description that applies to the interval.

The `DateStart` and `TimeStart`, and `DateEnd` and `TimeEnd` fields in the rate must be in UTC. Combining `DateStart` and `TimeStart` will yield a UTC datetime, as will combining `DateEnd` and `TimeEnd`. For example, for the first hour of March 1 in California (UTC-8), we would convert an interval start date of "2023-01-01" and start time of "00:00:00" to UTC, yielding a `DateStart` of "2023-01-01" and `TimeStart` of "08:00:00".

One day of a streaming rate would include the information Table 1. The table shows data for March 1, 2023 in the "America/Los_Angeles", also known as "PST/PDT" time zone. Note that the dates and times are all in UTC:

*Example* Hourly Rate Information for 2023-03-01 in California. Note that the dates and times are in UTC, not local time.

|DateStart|TimeStart|DateEnd|TimeEnd|DayStart|DayEnd|ValueName|Value|Unit|
|----------|--------|----------|--------|-|-|---------------|------|-----|
|2023-03-01|08:00:00|2023-03-01|08:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|09:00:00|2023-03-01|09:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|10:00:00|2023-03-01|10:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|11:00:00|2023-03-01|11:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|12:00:00|2023-03-01|12:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|13:00:00|2023-03-01|13:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|14:00:00|2023-03-01|14:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|15:00:00|2023-03-01|15:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|16:00:00|2023-03-01|16:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|17:00:00|2023-03-01|17:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|18:00:00|2023-03-01|18:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|19:00:00|2023-03-01|19:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|20:00:00|2023-03-01|20:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|21:00:00|2023-03-01|21:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|22:00:00|2023-03-01|22:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-01|23:00:00|2023-03-01|23:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-02|00:00:00|2023-03-01|00:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-02|01:00:00|2023-03-01|01:59:59|3|3|winter on peak|0.1388|$/kWh|
|2023-03-02|02:00:00|2023-03-01|02:59:59|3|3|winter on peak|0.1388|$/kWh|
|2023-03-02|03:00:00|2023-03-01|03:59:59|3|3|winter on peak|0.1388|$/kWh|
|2023-03-02|04:00:00|2023-03-01|04:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-02|05:00:00|2023-03-01|05:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-02|06:00:00|2023-03-01|06:59:59|3|3|winter off peak|0.1006|$/kWh|
|2023-03-02|07:00:00|2023-03-01|07:59:59|3|3|winter off peak|0.1006|$/kWh|

#### Rate Information

Uploaded rates must also contain rate information that makes up the header section of the XML or JSON. Many of these are optional, but including all of those applicable to the rate will substantially help users.

- **RateID** *required* This is the Rate Identification Number (RIN)
- **RateName** *required* The LSE’s name for the rate plan, consistent with the CEC’s Interval Meter Database as required by California Code of Regulations, Title 20, section 134¬4
- **RateType** *required* The applicable rate type; must be one of those in the RateType lookup table
- **AltRateName1** *optional* An alternative name for the rate
- **AltRateName2** *optional* A second alternative name for the rate
- **SignupCloseDate** *optional* The last day a customer may sign up for the rate
- **RatePlan_Url** *optional* A valid URL that directs to the utility webpage describing the rate plan
- **Sector** *optional* The sector that the rate applies to; must be one of those in the Sector lookup table
- **EndUse** *optional* The end use that the rate applies to; must be one of those in the EndUse lookup table
- **API_Url** *optional* A valid uniform resource locator (URL) that specifies the API that provides the values

## Registration & Authentication

> [!IMPORTANT]
> Registration and Authentication are only required for uploading to MIDAS.

All upload and access-management endpoints require registration and login to receive a **JWT Bearer token**. Tokens are valid for 60 minutes.

### Step 1 — Register

```text
POST /api/registration
```

All fields are Base64-encoded. Required fields: `fullname`, `username`, `password`, `emailaddress`, `organization`.

On success: HTTP 200, verification email sent to provided address. You must verify the email as explained in Step 2.

### Step 2 — Verify email

Once you get the confirmation code on your email, confirm programmatically:

```text
POST /api/confirmemail?username=<username>&confirmation_code=<code>
```

### Step 3 — Get a token

```text
GET /api/token
Authorization: Basic <base64(username:password)>
```

The JWT token is returned in the `Token` response **header**. The response body contains metadata (`token_type`, `expires_in`). Tokens expire after **3,600 seconds**.

### Step 4 — Request upload access (LSE upload access only)

```text
POST /api/uploadaccess/request
Authorization: Bearer <token>
Body: { "energy_code": "XX", "distribution_code": "XX" }
```

A CEC admin reviews and approves the request. You will receive an email confirmation when access is granted.

### Forgot password

```text
POST /api/forgotpassword?username=<username>
POST /api/confirmpasswordreset   { "token": "...", "new_password": "..." }
```

### Resend verification email

```text
POST /api/resendverification?username=<username>
```

## Uploading Rate Data to MIDAS (LSE Users)

```text
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

- Accepted intervals: `1 hour`, `15 minutes`, `5 minutes` only. All intervals for a unit must have the same duration.
- Interval assigned on first upload for a RIN — cannot change.
- All units in a single upload must have the same number of data points and the same interval.
- Missing values within a unit must be filled with zeros to maintain alignment.
- Uploading past-dated data is accepted but triggers a warning notification.
- A gap between the latest data in MIDAS and the earliest timestamp in your upload triggers a gap warning.
- MIDAS uses an upsert model: incoming datapoints are compared against existing records, inserting new intervals and updating any that already exist.

## Upload Rate Information

*This function is available to authorized LSE accounts only.*

Populating the RateInfo and Value tables requires a call to the ValueData endpoint. Upload data may be encoded in XML or JSON. Acceptable data entries are catalogued in supporting MIDAS lookup tables listed in the Appendix. At the time the rate data is uploaded, it is also added to the HistoricalData table and previous values in the Value table are deleted. For more information, see [Archiving Data to HistoricalData](README.md#archived-data-in-the-historicaldata-table) in the main documentation. Each RIN may only store a total of 50,000 values in the Value table.

### Prerequisites

1. Registered and email-verified account
2. Upload access approved by CEC for the relevant distribution and energy codes
3. Valid Bearer token (obtained via `GET /api/token`)

### POST ValueData

**Endpoint:** `/ValueData`

**HTTP Request:** `POST https://midasapi.energy.ca.gov/api/ValueData`

**Authorization:** Bearer

**Query Parameters:** None

**Response:** A successful upload will return an HTMLStatusCode of 200

#### POST Rate Python Example

```python
import os 
import sys
import requests

# File on the local filesystem that is correctly formatted against the XML schema definition (XSD) or analogously formatted JSON.
priceFileName = 'XML_streaming_upload.xml'

headers = {'accept': 'application/json', 'Content-Type': 'text/xml', 'Authorization': "Bearer " + token}
url = 'https://midasapi.energy.ca.gov/api/ValueData'
priceFile = open(priceFileName)
xml = priceFile.read()
priceFile.close()
pricing_response = requests.post(url, data = xml, headers = headers)

print(pricing_response.text)
```

## Tracking Upload Status — Jobs Endpoint

After a `POST /api/valuedata` call that passes initial validation, you receive a `jobID`. Poll the jobs endpoint to get the final validation result.

### How It Works

```text
POST /api/valuedata  →  HTTP 202 + jobID
         ↓
MIDAS backend runs full validation asynchronously
         ↓
GET /api/jobs/{jobID}  →  PROCESSING | COMPLETE | VALIDATION_FAILED
```

Job retention supports a configurable TTL (currently retained for 7 days), then automatically deleted.

### Get a Single Job

```text
GET /api/jobs/{jobID}
Authorization: Bearer <token>
```

**Status values:**

| Status | Meaning |
| ------ | ------- |
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

### List All Recent Jobs

```text
GET /api/jobs
Authorization: Bearer <token>
```

Optional query parameter: `detailed=true` to include per-RIN summaries in the list response. Results are returned newest-first.

### JobID Format

Job IDs are **[UUID version 7](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_7_(timestamp_and_random))** — time-sortable, globally unique identifiers. Example: `019e4c2d-6906-7cc4-a447-8259d107be54`.
