# Python Code Examples

## 1. Register a New Account

```python
import requests
import base64

BASE_URL = "https://midasapi.energy.ca.gov"

def b64(value: str) -> str:
    """Encode a string to base64."""
    return base64.b64encode(value.encode("utf-8")).decode("utf-8")

registration_payload = {
    "fullname": b64("Jane Smith"),
    "username": b64("jsmith_utility"),
    "password": b64("MySecureP@ss1"),
    "emailaddress": b64("jsmith@utility.com"),
    "organization": b64("Pacific Utility Co.")
}

response = requests.post(
    f"{BASE_URL}/api/registration",
    json=registration_payload,
    headers={"Content-Type": "application/json"}
)

print(response.status_code)   # 200
print(response.json())
# Check your email and click the verification link before continuing.
```

## 2. Confirm Email Programmatically

```python
# Use the code from your verification email
response = requests.post(
    f"{BASE_URL}/api/confirmemail",
    params={"username": "jsmith_utility", "confirmation_code": "123456"}
)
print(response.json())
```

## 3. Get a Token

```python
import requests
import base64

BASE_URL = "https://midasapi.energy.ca.gov"

def get_token(username: str, password: str) -> str:
    credentials = f"{username}:{password}"
    encoded = base64.b64encode(credentials.encode("utf-8")).decode("utf-8")
    
    response = requests.get(
        f"{BASE_URL}/api/token",
        headers={"Authorization": f"Basic {encoded}"}
    )
    response.raise_for_status()
    
    token = response.headers.get("Token")
    if not token:
        raise ValueError("Token not found in response headers")
    return token

token = get_token("jsmith_utility", "MySecureP@ss1")
print(f"Token acquired. Expires in 3600s.")
```

## 4. Get RIN List (No Auth Required in v2.0)

```python
import requests

BASE_URL = "https://midasapi.energy.ca.gov"

# Get all electricity rate RINs — no token needed
response = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={"SignalType": 1}
)
data = response.json()

# v2.0 response is wrapped in a keyed object
rin_list = data.get("Rates", [])
for rin in rin_list[:5]:
    print(rin["RateID"], "-", rin["Description"])
```

## 5. Query Flex Alert Data

```python
import requests

BASE_URL = "https://midasapi.energy.ca.gov"

# No auth needed for GET
response = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={
        "ID": "USCA-FLEX-ALRT-0000",
        "QueryType": "realtime"
    }
)
data = response.json()

# Each ValueInformation entry is one hour; Value=1 means active Flex Alert
for entry in data.get("ValueInformation", []):
    status = "ACTIVE" if entry["Value"] == 1 else "inactive"
    print(f"{entry['DateStart']} {entry['TimeStart']} — {status}")
```

## 6. Query GHG Emissions (SMUD Region)

```python
import requests

BASE_URL = "https://midasapi.energy.ca.gov"

response = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={
        "ID": "USCA-SGIP-MOER-SMUD",
        "QueryType": "realtime"
    }
)
data = response.json()

# Values are in g/kWh CO2 (NOT kg/kWh as in v1.0)
for entry in data.get("ValueInformation", [])[:5]:
    print(f"{entry['TimeStart']} — {entry['Value']} g/kWh CO2")
```

## 7. Upload Rate Data and Track Job Status

```python
import requests
import time
import json

BASE_URL = "https://midasapi.energy.ca.gov"

def upload_and_track(token: str, payload: dict) -> dict:
    """Upload rate data and poll until final status."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }

    # Step 1: Upload
    upload_response = requests.post(
        f"{BASE_URL}/api/valuedata",
        json=payload,
        headers=headers
    )
    upload_response.raise_for_status()
    
    result = upload_response.json()
    job_id = result["jobID"]
    print(f"Upload accepted. jobID: {job_id}")
    print(f"Track at: {result['statusUrl']}")

    # Step 2: Poll for completion
    max_attempts = 20
    poll_interval_seconds = 5

    for attempt in range(max_attempts):
        time.sleep(poll_interval_seconds)
        
        job_response = requests.get(
            f"{BASE_URL}/api/jobs/{job_id}",
            headers=headers
        )
        job_response.raise_for_status()
        job = job_response.json()
        
        status = job["status"]
        print(f"Attempt {attempt + 1}: {status}")
        
        if status == "COMPLETE":
            print("Upload validated successfully.")
            return job
        elif status == "VALIDATION_FAILED":
            print("Validation failed. Issues:")
            for rin in job.get("summary", {}).get("rins", []):
                for issue in rin.get("validationIssues", []):
                    print(f"  [{issue['type']}] {issue['code']}: {issue['message']}")
            return job

    raise TimeoutError(f"Job {job_id} did not complete within {max_attempts * poll_interval_seconds}s")


# Usage
token = get_token("jsmith_utility", "MySecureP@ss1")

with open("my_rate_upload.json") as f:
    payload = json.load(f)

final_result = upload_and_track(token, payload)
```

## 8. List Recent Upload Jobs

```python
import requests

BASE_URL = "https://midasapi.energy.ca.gov"

def list_my_jobs(token: str, detailed: bool = False) -> list:
    response = requests.get(
        f"{BASE_URL}/api/jobs",
        params={"detailed": str(detailed).lower()},
        headers={"Authorization": f"Bearer {token}"}
    )
    response.raise_for_status()
    data = response.json()
    return data.get("jobs", [])

token = get_token("jsmith_utility", "MySecureP@ss1")
jobs = list_my_jobs(token, detailed=True)

for job in jobs:
    rins = [r["rin"] for r in job.get("summary", {}).get("rins", [])]
    print(f"{job['id']} | {job['status']} | RINs: {rins}")
```

## 9. Get Historical Data for a Date Range

```python
import requests

BASE_URL = "https://midasapi.energy.ca.gov"

response = requests.get(
    f"{BASE_URL}/api/historicaldata/USCA-SGIP-MOER-PGE",
    params={
        "startdate": "2026-01-01",
        "enddate": "2026-01-31"
    }
)
data = response.json()
print(f"Returned {len(data.get('ValueInformation', []))} intervals")
```

## 10. Request Upload Access (LSE Self-Service)

```python
import requests

BASE_URL = "https://midasapi.energy.ca.gov"
token = get_token("jsmith_utility", "MySecureP@ss1")

response = requests.post(
    f"{BASE_URL}/api/uploadaccess/request",
    json={
        "energy_code": "PG",
        "distribution_code": "PG"
    },
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
)
print(response.json())
# You will receive an email when CEC approves the request.
```
