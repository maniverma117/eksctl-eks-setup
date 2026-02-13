# AWS Lambda → GCP BigQuery (Structured Logs via SQS)

This project implements a **cross-cloud logging pipeline** where:

**Java (Spring Boot)** → **AWS SQS** → **AWS Lambda (Python)** → **Google BigQuery**

The design supports **dynamic JSON payloads** safely without schema breakage.

---

## 🏗️ Architecture Overview

```
Spring Boot App
   |
   |  (JSON log message)
   v
AWS SQS
   |
   |  (batch event)
   v
AWS Lambda (Python 3.11)
   |
   |  (BigQuery insert)
   v
Google BigQuery
```

---

## 🧠 Key Design Decisions

* `jsonPayload` in BigQuery is a **RECORD**
* Dynamic JSON is wrapped under a **fixed sub-field**
* Prevents schema errors when payload changes
* Matches Google Cloud Logging semantics

---

## 🧾 BigQuery Table Schema (Required)

```text
severity            STRING
logName             STRING
resource            RECORD
jsonPayload         RECORD
  └── raw            STRING
timestamp           TIMESTAMP
receiveTimestamp    TIMESTAMP
textPayload         STRING
```

> ⚠️ `jsonPayload.raw` stores the full JSON as a string

---

## 🔐 Step 1: Create GCP Service Account

### 1. Create service account

```bash
gcloud iam service-accounts create bq-writer \
  --display-name "BigQuery Writer"
```

### 2. Grant permissions

```bash
gcloud projects add-iam-policy-binding <GCP_PROJECT_ID> \
  --member="serviceAccount:bq-writer@<GCP_PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataEditor"
```

```bash
gcloud projects add-iam-policy-binding <GCP_PROJECT_ID> \
  --member="serviceAccount:bq-writer@<GCP_PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser"
```

### 3. Create key

```bash
gcloud iam service-accounts keys create sa-key.json \
  --iam-account=bq-writer@<GCP_PROJECT_ID>.iam.gserviceaccount.com
```

---

## 🔐 Step 2: Prepare Key for AWS Lambda

### Base64 encode the key

```bash
base64 sa-key.json > gcp_sa_key.base64
```

### Add Lambda environment variable

```
GCP_SA_KEY_BASE64=<contents of gcp_sa_key.base64>
```

---

## 🐍 Step 3: Create Lambda Layer (Python 3.11)

### Use AWS Lambda base image (IMPORTANT)

```bash
docker run --rm -it \
  -v "$PWD":/var/task \
  public.ecr.aws/lambda/python:3.11 \
  bash
```

### Inside container

```bash
mkdir -p python
pip install google-cloud-bigquery -t python
zip -r gcp-bigquery-layer.zip python
```

### Publish layer

```bash
aws lambda publish-layer-version \
  --layer-name gcp-bigquery-layer \
  --zip-file fileb://gcp-bigquery-layer.zip \
  --compatible-runtimes python3.11
```

Attach this layer to your Lambda.

---

## ☕ Step 4: Java (Spring Boot) – Sending to SQS

### Java sends JSON wrapped under `raw`

```java
Map<String, Object> wrappedPayload = new HashMap<>();
wrappedPayload.put("raw", objectMapper.writeValueAsString(jsonMap));

queueMessage.put("jsonPayload", wrappedPayload);
```

This ensures compatibility with BigQuery `RECORD`.

---

## 🧠 Step 5: AWS Lambda Function (FINAL)

```python
import json
import boto3
from datetime import datetime, timezone
from google.cloud import bigquery
from google.oauth2 import service_account

# ---- CONFIG ----
SECRET_NAME = "gcp-sa-key"
REGION = "ap-south-1"

# cache for warm lambda
_cached_credentials = None
_cached_project = None
_bq_client = None


# ---------- SECRETS MANAGER ----------
def load_gcp_credentials():
    global _cached_credentials, _cached_project

    if _cached_credentials:
        return _cached_credentials, _cached_project

    sm = boto3.client("secretsmanager", region_name=REGION)
    response = sm.get_secret_value(SecretId=SECRET_NAME)

    secret = json.loads(response["SecretString"])

    # Fix private key newlines
    secret["private_key"] = secret["private_key"].replace("\\n", "\n")

    credentials = service_account.Credentials.from_service_account_info(secret)

    _cached_credentials = credentials
    _cached_project = secret["project_id"]

    return credentials, _cached_project


# ---------- BIGQUERY CLIENT ----------
def get_bq_client():
    global _bq_client

    if _bq_client:
        return _bq_client

    credentials, project = load_gcp_credentials()

    _bq_client = bigquery.Client(
        credentials=credentials,
        project=project,
    )

    return _bq_client


# ---------- HANDLER ----------
def lambda_handler(event, context):
    client = get_bq_client()

    rows_by_table = {}

    for record in event["Records"]:
        body = json.loads(record["body"])

        # Dynamic table
        project_id = body["project_id"]
        dataset_id = body["dataset_id"]
        table_id = body["table_id"]
        full_table_id = f"{project_id}.{dataset_id}.{table_id}"

        # Row schema
        row = {
            "severity": body.get("severity"),
            "logName": body.get("logName"),
            "resource": body.get("resource"),
            "jsonPayload": body.get("jsonPayload"),
            "timestamp": body.get("timestamp"),
            "receiveTimestamp": datetime.now(timezone.utc).isoformat(),
            "textPayload": None
        }

        rows_by_table.setdefault(full_table_id, []).append(row)

    # Insert rows
    total = 0
    for table, rows in rows_by_table.items():
        errors = client.insert_rows_json(table, rows)

        if errors:
            print(f"BigQuery insert errors for {table}: {errors}")
            raise Exception("BigQuery insert failed")

        total += len(rows)

    return {
        "statusCode": 200,
        "rows_inserted": total
    }
```

---

## 🔍 Querying in BigQuery

```sql
SELECT
  JSON_VALUE(jsonPayload.raw, '$.account_slug') AS account,
  JSON_VALUE(jsonPayload.raw, '$.vendorOrderNumber') AS order_no,
  timestamp
FROM `project.dataset.inventory_updates`
ORDER BY timestamp DESC;
```

---

## 🚨 Common Errors & Fixes

| Error                             | Cause                    | Fix                   |
| --------------------------------- | ------------------------ | --------------------- |
| `Cannot convert string to record` | Sending STRING to RECORD | Wrap JSON under `raw` |
| `No module google.protobuf`       | Wrong Lambda build env   | Use Lambda base image |
| `KeyError GCP_SA_KEY_BASE64`      | Missing env var          | Add to Lambda         |
| Schema breaks                     | Dynamic JSON             | Use `raw STRING`      |

---

## ✅ Final Status

✔ Java → SQS working
✔ Lambda authentication working
✔ Lambda layer correct
✔ BigQuery inserts stable
✔ Schema future-proof

---

## 🏁 Conclusion

This setup is **production-grade**, **schema-safe**, and **cloud-native**.

You now have:

* Zero schema churn
* Safe cross-cloud logging
* Full JSON preserved
* Easy querying
