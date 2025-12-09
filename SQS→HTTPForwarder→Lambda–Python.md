````markdown
# SQS → HTTP Forwarder (AWS Lambda – Python)

This Lambda function reads messages from an **SQS queue** and forwards the **raw SQS body** as a JSON payload to a target HTTP/HTTPS endpoint.

It is a Python equivalent of the original Node.js implementation using `fetch`.

---

## 🧩 Use Case

- Application A publishes events/messages to **SQS**
- This Lambda is subscribed as an **SQS trigger**
- For each SQS message:
  - It reads `body` (JSON string)
  - Parses it to find:
    - `url` – destination endpoint
    - `httpMethod` (optional, default: `POST`)
  - Sends the **exact same body** as the HTTP request payload

---

## 🗂 Payload Format

Each SQS message **body** must be a JSON string with at least:

```json
{
  "url": "https://example.com/webhook",
  "httpMethod": "POST",
  "someField": "someValue",
  "nestedData": {
    "foo": "bar"
  }
}
````

* `url` – **required**, target endpoint
* `httpMethod` – optional, defaults to `"POST"`
* Any other fields are included unchanged in the forwarded body.

The Lambda forwards the **raw SQS body string** as the HTTP request body.

---

## 🧪 Sample SQS Event

Example SQS event that Lambda receives from AWS:

```json
{
  "Records": [
    {
      "messageId": "11111111-2222-3333-4444-555555555555",
      "receiptHandle": "AQEB...",
      "body": "{\"url\":\"https://example.com/webhook\",\"httpMethod\":\"POST\",\"foo\":\"bar\"}",
      "attributes": {},
      "messageAttributes": {},
      "md5OfBody": "string",
      "eventSource": "aws:sqs",
      "eventSourceARN": "arn:aws:sqs:region:account-id:queue-name",
      "awsRegion": "region"
    }
  ]
}
```

---

## 🧾 Lambda Handler (Python)

```python
import json
import urllib3

http = urllib3.PoolManager()

def handler(event, context):
    for record in event.get("Records", []):
        try:
            # Raw SQS body (string)
            raw_body = record["body"]

            # Parse once to read url + method
            body_obj = json.loads(raw_body)

            url = body_obj.get("url")
            method = body_obj.get("httpMethod", "POST")

            print(f"Forwarding full data to: {url}")
            print(f"Method: {method}")

            # Forward EXACT same raw body
            response = http.request(
                method,
                url,
                body=raw_body.encode("utf-8"),
                headers={"Content-Type": "application/json"}
            )

            print("Response status:", response.status)
            print("Response body:", response.data.decode("utf-8"))

        except Exception as e:
            print("Error:", str(e))
            # Uncomment if you want message to be retried by SQS:
            # raise e

    return {"statusCode": 200}
```

---

## 📦 Dependencies

* **Runtime**: Python 3.10 (or 3.9/3.11 etc.)
* **Library**: `urllib3` – **already included** in AWS Lambda Python runtime.
  👉 No Lambda layer is required.

You can just use:

```python
import urllib3
```

directly.

---

## 🔐 IAM Permissions

The Lambda needs:

* Permission to be invoked by SQS:

  * Usually handled automatically when you configure the **SQS trigger** in the console or via CloudFormation/Terraform.
* It does **not** need special IAM permissions to make outbound HTTP calls (that happens via the VPC’s internet route / NAT, if configured).

Example minimal execution role (conceptual):

* `AWSLambdaBasicExecutionRole` (for CloudWatch Logs)
* `Allow` SQS trigger source mapping (created automatically)

---

## ⚙️ Configuration & Deployment

### 1. Create SQS Queue

* Create an SQS queue (standard or FIFO as needed).
* Ensure your producer sends messages where `body` is a JSON string including `url`.

### 2. Create Lambda Function

* Runtime: **Python 3.x**
* Handler: `index.handler` (if file is `index.py`)
* Paste the handler code above into `index.py`.

### 3. Add SQS Trigger

* In Lambda console:

  * Add **SQS** as a trigger
  * Select your queue
  * Configure:

    * **Batch size** (e.g., 1–10 messages per invocation)
    * **Maximum retry attempts** (Optional, depends on design)
    * **Dead Letter Queue (DLQ)** (recommended)

### 4. Networking

If the target `url` is:

* **Public internet**: ensure Lambda has NAT or internet access (if in a VPC) or no VPC required.
* **Private endpoint**: ensure Lambda’s VPC configuration, security groups, and routing allow access to the destination.

---

## 🔁 Error Handling & Retries

* On any error in processing a record:

  * The code logs the error with `print("Error:", str(e))`.
  * If you **uncomment** `raise e`, the whole batch will fail.
  * SQS will **retry** delivery according to:

    * SQS redrive policy / max receive count
    * Lambda event source mapping config
* For production:

  * Consider **enabling DLQ (Dead Letter Queue)** for failed messages.
  * Optionally, add more detailed logging for response status codes (e.g., treat 4xx vs 5xx differently).

---

## ✅ Behavior Summary

* Reads all records from the `event.Records` array.
* For each record:

  * Parses `record.body` JSON
  * Extracts `url` and `httpMethod`
  * Sends HTTP request with:

    * Method: `httpMethod` or `POST`
    * Headers: `Content-Type: application/json`
    * Body: **exact original SQS `body` string**
* Logs response status and body to CloudWatch Logs.
* Returns `{ "statusCode": 200 }` after processing all records.

---

## 🧱 Extending This Function

You can extend this Lambda to:

* Add **authentication headers** (API keys, bearer tokens, etc.)
* Implement **per-message routing logic** (different URLs per message type)
* Validate `url` against an allow-list to avoid misuse
* Add **metrics** (e.g., via CloudWatch Embedded Metrics) for success/failure counts

---

```
```
