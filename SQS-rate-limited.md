# 🕹️ SQS Rate Limiting Patterns using EventBridge and Lambda

This project demonstrates two common patterns to **control the rate of SQS usage** in AWS using EventBridge, Lambda, and reserved concurrency.


## 📦 Patterns Covered

### 🅰️ Pattern A: EventBridge Scheduler → Lambda → SQS (Controlled Push Rate)
- Use **EventBridge scheduled rule** to invoke a Lambda every `N` minutes.
- Lambda pushes a controlled number of messages to an SQS queue.
- Useful when you want to **limit how often messages are sent to SQS**.

### 🅱️ Pattern B: SQS → Lambda (Controlled Pull Rate via Reserved Concurrency)
- Lambda is triggered by SQS.
- Processing is **rate-limited by setting reserved concurrency**.
- Useful when you want to **limit how fast messages are consumed** from the queue.

---

## 🔧 Prerequisites

- AWS CLI configured (`aws configure`)
- IAM role with necessary permissions for Lambda, SQS, EventBridge
- Python 3.11 or higher

---

## 🅰️ Pattern A: Scheduled Push to SQS

### 1. Create the SQS Queue
```bash
aws sqs create-queue --queue-name MyRateLimitedQueue
````

### 2. Create the Lambda Function

**lambda\_function.py**

```python
import boto3
import os

sqs = boto3.client('sqs')
queue_url = os.environ['QUEUE_URL']

def lambda_handler(event, context):
    for i in range(5):  # Controlled push rate
        sqs.send_message(
            QueueUrl=queue_url,
            MessageBody=f"Message {i + 1} from Lambda"
        )
    return {"status": "Messages sent"}
```

```bash
zip function.zip lambda_function.py

aws lambda create-function \
  --function-name SQSMessageSender \
  --zip-file fileb://function.zip \
  --handler lambda_function.lambda_handler \
  --runtime python3.11 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/<LambdaExecutionRole> \
  --environment Variables="{QUEUE_URL=https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/MyRateLimitedQueue}"
```

### 3. Create EventBridge Rule

```bash
aws events put-rule \
  --schedule-expression "rate(1 minute)" \
  --name RateLimitScheduler
```

### 4. Allow EventBridge to Trigger Lambda

```bash
aws lambda add-permission \
  --function-name SQSMessageSender \
  --statement-id EventBridgeInvoke \
  --action 'lambda:InvokeFunction' \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:<REGION>:<ACCOUNT_ID>:rule/RateLimitScheduler
```

### 5. Add Lambda as Rule Target

```bash
aws events put-targets \
  --rule RateLimitScheduler \
  --targets "Id"="1","Arn"="arn:aws:lambda:<REGION>:<ACCOUNT_ID>:function:SQSMessageSender"
```

---

## 🅱️ Pattern B: Rate-Limited SQS Consumption

### 1. Create the SQS Queue

```bash
aws sqs create-queue --queue-name MyProcessingQueue
```

### 2. Create Lambda Function

**lambda\_function.py**

```python
def lambda_handler(event, context):
    for record in event['Records']:
        print("Processing message:", record['body'])
    return {"status": "processed"}
```

```bash
zip function.zip lambda_function.py

aws lambda create-function \
  --function-name SQSLimitedProcessor \
  --zip-file fileb://function.zip \
  --handler lambda_function.lambda_handler \
  --runtime python3.11 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/<LambdaExecutionRole>
```

### 3. Add SQS as Event Source

```bash
aws lambda create-event-source-mapping \
  --function-name SQSLimitedProcessor \
  --batch-size 1 \
  --event-source-arn arn:aws:sqs:<REGION>:<ACCOUNT_ID>:MyProcessingQueue
```

### 4. Set Reserved Concurrency (to limit processing rate)

```bash
aws lambda put-function-concurrency \
  --function-name SQSLimitedProcessor \
  --reserved-concurrent-executions 1
```

---

## 🔐 IAM Policy Requirements

The Lambda execution role should have:

```json
{
  "Effect": "Allow",
  "Action": [
    "sqs:SendMessage",
    "sqs:ReceiveMessage",
    "sqs:DeleteMessage"
  ],
  "Resource": "arn:aws:sqs:<REGION>:<ACCOUNT_ID>:*"
}
```

---

## ✅ Output

* Pattern A: Sends 5 messages every minute to SQS.
* Pattern B: Processes 1 message at a time from SQS (rate-limited).

---

## 📚 Further Ideas

* Add CloudWatch alarms for dead-letter queue usage.
* Extend Lambda with exponential backoff or SQS delaySeconds for smarter throttling.
* Use Step Functions for more complex flow control.

---

## 🧼 Cleanup

```bash
aws lambda delete-function --function-name SQSMessageSender
aws lambda delete-function --function-name SQSLimitedProcessor
aws sqs delete-queue --queue-url <queue-url>
aws events delete-rule --name RateLimitScheduler
```

---

## 👨‍💻 Author

Mani V — DevOps Engineer | AWS | Automation | RAG Systems

```

