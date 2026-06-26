# Watttime's SGIP GHG and CAISO's FlexAlert Signals in MIDAS

## Table of Contents

1. [SGIP GHG Emissions RIN Consolidation: `USCA-SGIP-MOER-{REGION}`](#1-sgip-ghg-emissions-rin-consolidation-usca-sgip-moer-region)
2. [Flex Alert RIN Consolidation: `USCA-FLEX-ALRT-0000`](#2-flex-alert-rin-consolidation-usca-flex-alrt-0000)

## 1. SGIP GHG Emissions RIN Consolidation: `USCA-SGIP-MOER-{REGION}`

**Component:** MIDAS API / ValueData & HistoricalData
**Related Repo:** [california-energy-commission/MIDAS](https://github.com/california-energy-commission/MIDAS)

Today, MIDAS exposes SGIP GHG emissions data through **33 separate RINs** — real-time, forecast, and historical variants for each of 11 California grid regions. Consumers have had to call different RINs depending on which time window they needed, and the real-time RIN returned only a single data point rather than a usable time series.

We're replacing all 33 RINs with **11 region-specific RINs** — one per grid region — each delivering GHG emissions as a continuous, predictable 5-minute signal using the same endpoint contract as all other MIDAS rate signals.

### ⚠️ Breaking Change: Unit of Measure — `kg/kWh CO2` → `g/kWh CO2`

As part of this update, the unit for all SGIP GHG emissions values is changing from **kilograms per kilowatt-hour** (`kg/kWh CO2`) to **grams per kilowatt-hour** (`g/kWh CO2`).

| | Previous | New |
| --- | --- | --- |
| `Unit` field | `kg/kWh CO2` | `g/kWh CO2` |
| Example value | `0.345` | `345.0` |
| Scale factor | — | × 1,000 |

**Action required for all consumers:** Any application that reads the `Value` field from SGIP GHG RINs must update its logic to account for the new scale. Comparisons, thresholds, charts, and stored data based on the old kg values will be incorrect without this update.

### Background / Previous Behavior

MIDAS previously included **three RIN families** for SGIP GHG data, each spanning 11 regions (33 RINs total):

| RIN Pattern | Code Meaning | Behavior |
| --- | --- | --- |
| `USCA-SGIP-SGRT-XXXX` | **SGRT** = SGIP Real-Time | Returned **only one data point** — the current CO2 level for the region at query time |
| `USCA-SGIP-SGFC-XXXX` | **SGFC** = SGIP Forecast | Returned forecasted CO2 levels at 5-minute intervals from the WattTime API |
| `USCA-SGIP-SGHT-XXXX` | **SGHT** = SGIP Historic | Returned previously observed values; once a real-time value was superseded, it was moved to the `HistoricalData` table and retrievable here |

All data originated from the [WattTime SGIP Signal API](https://sgipsignal.com/). Consumers had to combine calls across all three RIN families to assemble a complete past-through-future emissions picture for any region.

> [!NOTE]
> Due to GHG RIN changes from MIDAS 1.0 to 2.0, historical GHG data prior to June 2026 was not migrated and is no longer available in MIDAS. All historical SGIP data is available in CSV format from <https://content.sgipsignal.com/download-data/>

### What End Users Will See

#### One RIN per region, two query types — same as every other MIDAS rate signal

| Question the consumer is asking | API call |
| --- | --- |
| What are the GHG emissions for today through the next two days? | `GET /api/ValueData?ID=USCA-SGIP-MOER-{REGION}&QueryType=realtime` |
| What were the GHG emissions over the past 90 days? | `GET /api/ValueData?ID=USCA-SGIP-MOER-{REGION}&QueryType=alldata` |
| What were the GHG emissions for a specific time range? | `GET /api/HistoricalData?ID=USCA-SGIP-MOER-{REGION}&startdate={date}&enddate={date}` |

#### 5-Minute continuous intervals

Each response contains **one row per 5-minute interval**, with the CO2 emissions value for that period in `g/kWh CO2`. The `realtime` response seamlessly blends observed historical data, the latest real-time reading, and WattTime forecast data into a single unbroken time series.

### Previous RINs Retired

| Retired RIN Pattern | Recommended Replacement |
| --- | --- |
| `USCA-SGIP-SGRT-XXXX` | `USCA-SGIP-MOER-{REGION}` with `QueryType=realtime` |
| `USCA-SGIP-SGFC-XXXX` | `USCA-SGIP-MOER-{REGION}` with `QueryType=realtime` |
| `USCA-SGIP-SGHT-XXXX` | `USCA-SGIP-MOER-{REGION}` with `QueryType=alldata` or `/api/HistoricalData` |

### New RINs: `USCA-SGIP-MOER-{REGION}`

**MOER** = Marginal Operating Emissions Rate — the standard term for the grid-marginal GHG signal provided by WattTime.

#### Region Reference Table and MIDAS 1.0 -> 2.0 RIN Mapping

| Utility / Region | WattTime `ba` Code | MIDAS 2.0 RIN | MIDAS 1.0 RINs |
| --- | --- | --- | --- |
| PacifiCorp West | `SGIP_PACW` | `USCA-SGIP-MOER-PACW` | `USCA-SGIP-SGRT-PACW`, `USCA-SGIP-SGFC-PACW`, `USCA-SGIP-SGHT-PACW` |
| CAISO – San Diego Gas & Electric | `SGIP_CAISO_SDGE` | `USCA-SGIP-MOER-SDGE` | `USCA-SGIP-SGRT-SDGE`, `USCA-SGIP-SGFC-SDGE`, `USCA-SGIP-SGHT-SDGE` |
| CAISO – Pacific Gas & Electric | `SGIP_CAISO_PGE` | `USCA-SGIP-MOER-PGE` | `USCA-SGIP-SGRT-PGE`, `USCA-SGIP-SGFC-PGE`, `USCA-SGIP-SGHT-PGE` |
| BANC Public Authority 2 | `SGIP_BANC_P2` | `USCA-SGIP-MOER-P2` | `USCA-SGIP-SGRT-BANC`, `USCA-SGIP-SGFC-BANC`, `USCA-SGIP-SGHT-BANC` |
| Turlock Irrigation District | `SGIP_TID` | `USCA-SGIP-MOER-TID` | `USCA-SGIP-SGRT-TID`, `USCA-SGIP-SGFC-TID`, `USCA-SGIP-SGHT-TID` |
| BANC – Sacramento Municipal Utility District | `SGIP_BANC_SMUD` | `USCA-SGIP-MOER-SMUD` | `USCA-SGIP-SGRT-SMUD`, `USCA-SGIP-SGFC-SMUD`, `USCA-SGIP-SGHT-SMUD` |
| Imperial Irrigation District | `SGIP_IID` | `USCA-SGIP-MOER-IID` | `USCA-SGIP-SGRT-IID`, `USCA-SGIP-SGFC-IID`, `USCA-SGIP-SGHT-IID` |
| NV Energy | `SGIP_NVENERGY` | `USCA-SGIP-MOER-NVENERGY` | `USCA-SGIP-SGRT-NVENERGY`, `USCA-SGIP-SGFC-NVENERGY`, `USCA-SGIP-SGHT-NVENERGY` |
| Western Area Lower Colorado | `SGIP_WALC` | `USCA-SGIP-MOER-WALC` | `USCA-SGIP-SGRT-WALC`, `USCA-SGIP-SGFC-WALC`, `USCA-SGIP-SGHT-WALC` |
| CAISO – Southern California Edison | `SGIP_CAISO_SCE` | `USCA-SGIP-MOER-SCE` | `USCA-SGIP-SGRT-SCE`, `USCA-SGIP-SGFC-SCE`, `USCA-SGIP-SGHT-SCE` |
| Los Angeles Dept. of Water & Power | `SGIP_LADWP` | `USCA-SGIP-MOER-LADWP` | `USCA-SGIP-SGRT-LADWP`, `USCA-SGIP-SGFC-LADWP`, `USCA-SGIP-SGHT-LADWP` |

For the full region list and definitions, see [sgipsignal.com/grid-regions](https://sgipsignal.com/grid-regions).

#### RIN Metadata

| Field | Value |
| --- | --- |
| RateID | `USCA-SGIP-MOER-{REGION}` |
| RateName | `{ba_code} Realtime GHG Emissions` (e.g. `SGIP_BANC_SMUD Realtime GHG Emissions`) |
| RateType | `Greenhouse Gas emissions` |
| SignalType | `Greenhouse Gas Emissions` |
| Unit | `g/kWh CO2` |
| ValueName | `Realtime SGIP GHG Emission` |
| Granularity | 5-minute intervals |
| Value | Float — CO2 emissions in g per kWh |

### Request Examples

#### Real-time (72-hour window, 5-minute intervals)

```http
GET /api/ValueData?ID=USCA-SGIP-MOER-SMUD&QueryType=realtime
Authorization: Bearer <jwt>
```

Returns 5-minute ValueInformation entries covering a 72-hour window that opens at midnight Pacific on the request date and closes at 23:59:59 Pacific on Day+2 (approximately 864 intervals, ±1 on DST transition days). All DateStart, DateEnd, TimeStart, and TimeEnd fields in the response are expressed in UTC, consistent with the standard MIDAS response contract.

#### All-data (past 90 days, 5-minute intervals)

```http
GET /api/ValueData?ID=USCA-SGIP-MOER-SMUD&QueryType=alldata
Authorization: Bearer <jwt>
```

Returns approximately **25,920 `ValueInformation` entries** (90 days × 24 hours × 12 intervals/hour; ±1 interval on DST transition days). The window ends at 23:59:59 Pacific on Day+2, matching the real-time window boundary. All DateStart, DateEnd, TimeStart, and TimeEnd fields in the response are expressed in UTC, consistent with the standard MIDAS response contract.

#### All-data (past 90 days, 5-minute intervals)

#### Historical (custom date range)

```http
GET /api/HistoricalData?ID=USCA-SGIP-MOER-SMUD&startdate=2026-04-01&enddate=2026-04-07
Authorization: Bearer <jwt>
```

Returns 5-minute intervals between `startdate` and `enddate`, inclusive.

### Response Shape

Identical to every other rate signal in MIDAS — there are **no SGIP-specific fields**.

```json
{
  "RateID": "USCA-SGIP-MOER-SMUD",
  "SystemTime_UTC": "2026-04-20T23:15:00.544586Z",
  "RateName": "SGIP_BANC_SMUD Realtime GHG Emissions",
  "RateType": "Greenhouse Gas emissions",
  "Sector": null,
  "API_Url": "https://sgipsignal.com",
  "RatePlan_Url": null,
  "EndUse": null,
  "AltRateName1": null,
  "AltRateName2": null,
  "SignupCloseDate": null,
  "SignalType": "Greenhouse Gas Emissions",
  "Description": "Realtime WattTime SGIP Green House Gas (GHG) Emission Signals",
  "ValueInformation": [
    {
      "ValueName": "Realtime SGIP GHG Emission",
      "DateStart": "2026-04-20",
      "DateEnd": "2026-04-20",
      "DayStart": 1,
      "DayEnd": 1,
      "TimeStart": "07:00:00",
      "TimeEnd": "07:04:59",
      "Unit": "g/kWh CO2",
      "Value": 333.8
    },
    {
      "ValueName": "Realtime SGIP GHG Emission",
      "DateStart": "2026-04-20",
      "DateEnd": "2026-04-20",
      "DayStart": 1,
      "DayEnd": 1,
      "TimeStart": "07:05:00",
      "TimeEnd": "07:09:59",
      "Unit": "g/kWh CO2",
      "Value": 332.5
    }
    // ... 5-minute intervals continue through Day+2 Pacific 23:59:59 (the date and time in the response are in UTC)
  ]
}
```

**Notes for consumers:**

- `DateStart` / `TimeStart` reflect UTC per the standard MIDAS contract for all RINs.
- `Value` is a float representing CO2 in g per kWh.
- `ValueName` is always `"Realtime SGIP GHG Emission"` regardless of whether the interval is historical, real-time, or forecast.
- `DayStart` / `DayEnd` follow the same day-of-week numbering convention used by other MIDAS rate signals (1 = Monday through 7 = Sunday; 8 = Holiday).

### Error Responses

| Code | Condition |
| --- | --- |
| `400 Bad Request` | Invalid `QueryType` value or unrecognized RIN |

## 2. Flex Alert RIN Consolidation: `USCA-FLEX-ALRT-0000`

**Component:** MIDAS API / ValueData & HistoricalData 

**Related Repo:** [california-energy-commission/MIDAS](https://github.com/california-energy-commission/MIDAS) 

Today, MIDAS exposes Flex Alert information through **three separate Rate Identification Numbers (RINs)** — one for real-time status, one for forecasts, and one for historical data (the historical one was never fully functional and currently returns an error). Consumers integrating Flex Alert data have had to call different RINs for different time windows and parse a custom response format that doesn't match how other MIDAS rate signals work.

We're replacing the three RINs with **one unified RIN** that delivers Flex Alert status as a simple, predictable hourly signal — the same shape as electricity rates and greenhouse-gas emissions.

### Background / Previous Behavior

MIDAS previously included **three separate Flex Alert RINs**:

| RIN | Code Meaning | Behavior |
| --- | ------------ | -------- |
| `USCA-FLEX-FXRT-0000` | **FXRT** = Flex Alert Real-Time | Checked CAISO Flex Alert status at query time; result passed directly to caller |
| `USCA-FLEX-FXFC-0000` | **FXFC** = Flex Alert Forecasted | Checked CAISO for planned/forecasted Flex Alerts at query time; result passed directly to caller |
| `USCA-FLEX-FXHT-0000` | **FXHT** = Flex Alert Historical | Intended to return all previously active Flex Alerts; **never fully functional — currently returns an error** |

The California ISO (CAISO) maintains information on both active and planned Flex Alerts. MIDAS polled the CAISO Flex Alert webpage and persisted that data into the database. The `FXRT` and `FXFC` RINs acted as real-time pass-throughs; `FXHT` was designed to serve the full historical corpus but was never operational.

### What End Users Will See

#### One RIN, two query types — same as every other MIDAS rate signal

| Question the consumer is asking | API call |
| --- | --- |
| Is a Flex Alert active right now? | `GET /api/ValueData?ID=USCA-FLEX-ALRT-0000&QueryType=realtime` |
| What's the history of Flex Alerts over the past 90 days? | `GET /api/ValueData?ID=USCA-FLEX-ALRT-0000&QueryType=alldata` |

#### Hourly binary status

Each response contains **one row per hour**, with two fixed shapes:

- `Value = 1`, `ValueName = "Active Flex Alert"` — a Flex Alert was active during that hour
- `Value = 0`, `ValueName = "No Active Flex Alert"` — no Flex Alert was active during that hour

No gaps in the response: every hour in the requested window appears exactly once.

### Previous RINs Will Retire

| Deprecated RIN | Recommended Replacement |
| -------------- | ----------------------- |
| `USCA-FLEX-FXRT-0000` | `USCA-FLEX-ALRT-0000` with `QueryType=realtime` |
| `USCA-FLEX-FXFC-0000` | `USCA-FLEX-ALRT-0000` with `QueryType=realtime` |
| `USCA-FLEX-FXHT-0000` | `USCA-FLEX-ALRT-0000` with `QueryType=alldata` |

### New RIN: `USCA-FLEX-ALRT-0000`

| Field | Value |
| ----- | ----- |
| RateID | `USCA-FLEX-ALRT-0000` |
| RateName | `CAISO Flex Alert Status` |
| RateType | `Flex Alert` |
| SignalType | `California Independent System Operator Flex Alert` *(integer `3` in `RateInfo` queries)* |
| Unit | `Event` |
| ValueName (active hour) | `Active Flex Alert` |
| ValueName (inactive hour) | `No Active Flex Alert` |
| Value | `1` (active) or `0` (no active alert) |

### Request Examples

#### Real-time (72 hours, hourly)

```http
GET /api/ValueData?ID=USCA-FLEX-ALRT-0000&QueryType=realtime
Authorization: Bearer <jwt>
```

Returns **72 `ValueInformation` entries**, one per Pacific hour, anchored at Pacific midnight on the day of the request — covering today through the end of the day two days from now. The date and time are in UTC.

#### All-data (90 days, hourly)

```http
GET /api/ValueData?ID=USCA-FLEX-ALRT-0000&QueryType=alldata
Authorization: Bearer <jwt>
```

Returns approximately **2,160 `ValueInformation` entries** (90 calendar days × 24 hours; ±1 hour on Daylight Saving Time transition days). The window ends at the same boundary as the real-time response — the end of day two days from the request date in Pacific time.

### Response Shape

Identical to every other rate signal in MIDAS — there are **no Flex Alert-specific fields**.

```json
{
  "RateID": "USCA-FLEX-ALRT-0000",
  "SystemTime_UTC": "2026-05-07T18:00:00Z",
  "RateName": "CAISO Flex Alert Status",
  "RateType": "Flex Alert",
  "Sector": null,
  "API_Url": null,
  "RatePlan_Url": null,
  "EndUse": null,
  "AltRateName1": null,
  "AltRateName2": null,
  "SignupCloseDate": null,
  "SignalType": "California Independent System Operator Flex Alert",
  "Description": "CAISO Flex Alert Status - Flex Alert",
  "ValueInformation": [
    {
      "ValueName": "No Active Flex Alert",
      "DateStart": "2026-05-07",
      "DateEnd": "2026-05-07",
      "DayStart": 4,
      "DayEnd": 4,
      "TimeStart": "00:00:00",
      "TimeEnd": "00:59:59",
      "Unit": "Event",
      "Value": 0
    },
    {
      "ValueName": "Active Flex Alert",
      "DateStart": "2026-05-07",
      "DateEnd": "2026-05-07",
      "DayStart": 4,
      "DayEnd": 4,
      "TimeStart": "01:00:00",
      "TimeEnd": "01:59:59",
      "Unit": "Event",
      "Value": 1
    },
...
  ]
}
```

**Notes for consumers:**

- `DateStart` / `TimeStart` reflect UTC per the standard MIDAS contract for all RINs.
- `Value` is always `0` or `1`. No partial values. No null values.
- The response contains **no gaps**: every hour in the requested window appears exactly once.
- `DayStart` / `DayEnd` follow the same day-of-week numbering convention used by other MIDAS rate signals (1 = Monday through 7 = Sunday; 8 = Holiday).

### Motivation

- **Reduce client complexity** caused by having to choose among three RINs for a single underlying signal.
- **Align with MIDAS 1.5 goals** — performance, reliability, and bounded payloads.
- **Provide a consistent time-series shape** for Flex Alert data that matches all other MIDAS rate signals.
- **Limit unbounded downloads** and improve API responsiveness by capping alldata windows at 90 days.

### Error Responses

| Code | Condition |
|------|-----------|
| `400 Bad Request` | Invalid `QueryType` value or unrecognized RIN |
