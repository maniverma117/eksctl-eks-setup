# 📦 Helm Chart Deployment via Amazon S3

This project demonstrates how to host and install Helm charts using an Amazon S3 bucket as a central repository, powered by the `helm-s3` plugin.


## ✅ Prerequisites

Before you begin, ensure the following are set up:

### 1. AWS CLI
Install and configure the AWS CLI:

```bash
aws configure
````

Make sure the credentials used have access to the target S3 bucket.

### 2. S3 Bucket

Create a dedicated S3 bucket (e.g., `velocis-helm`) to serve as the central Helm chart repository:

```bash
aws s3 mb s3://velocis-helm
```

### 3. S3 Bucket Policy (Optional: Public Read Access)

To allow public access to the chart files (if required):

```json
{
  "Version":"2012-10-17",
  "Statement":[{
    "Sid":"PublicReadGetObject",
    "Effect":"Allow",
    "Principal": "*",
    "Action":["s3:GetObject"],
    "Resource":["arn:aws:s3:::velocis-helm/*"]
  }]
}
```

Apply the policy:

```bash
aws s3api put-bucket-policy --bucket velocis-helm --policy file://bucket-policy.json
```

---

## 🧰 Setup Steps

### 1. Install `helm-s3` Plugin

```bash
helm plugin install https://github.com/hypnoglow/helm-s3.git
```

### 2. Initialize the Helm S3 Repository (only once)

```bash
helm s3 init s3://velocis-helm
```

### 3. Package Your Helm Chart

```bash
helm package ./velocis
```

### 4. Add the S3 Repo Locally

```bash
helm repo add velocis s3://velocis-helm
```

### 5. Push Chart to S3

```bash
helm s3 push velocis-<version>.tgz velocis
```

> Replace `<version>` with your actual chart version.

### 6. Update Repositories

```bash
helm repo update
```

---

## 🚀 Install Helm Chart

Dry run:

```bash
helm install test velocis/velocis --dry-run
```

Actual install:

```bash
helm install test velocis/velocis
```

---

## 📌 Notes

* To update an existing chart version, bump the version in `Chart.yaml`.
* Use versioning properly to prevent caching/stale index issues.
* For CI/CD, ensure your runner has AWS CLI configured and the `helm-s3` plugin installed.

---

## 📂 Resources

* Helm S3 Plugin: [https://github.com/hypnoglow/helm-s3](https://github.com/hypnoglow/helm-s3)
* Helm Docs: [https://helm.sh/docs](https://helm.sh/docs)
* AWS CLI Docs: [https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

