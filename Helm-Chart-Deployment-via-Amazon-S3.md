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




# ===============================================================================================

# 📦 Helm Chart Deployment via GitHub Pages

This project demonstrates how to host and install Helm charts using a **GitHub repository** (via GitHub Pages) as a central Helm chart repository.


## ✅ Prerequisites

Before you begin, ensure the following are set up:

### 1. Helm CLI

Install Helm if not already installed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
````

Verify installation:

```bash
helm version
```

### 2. GitHub Repository

Create a GitHub repository named (e.g., `velocis-helm-charts`) where your Helm chart(s) will be hosted.

> Enable GitHub Pages for this repo:
>
> * Go to **Settings > Pages**
> * Source: `main` branch (or `gh-pages` if you use a separate branch)
> * Folder: `/ (root)`
> * Save and note the URL, e.g., `https://<username>.github.io/velocis-helm-charts`

### 3. Chart Directory

Ensure your Helm chart is in a valid structure:

```
.
├── charts/
│   └── velocis/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
```

---

## 🧰 Setup Steps

### 1. Package Your Helm Chart

Navigate to the directory where your chart is located:

```bash
cd charts/velocis
helm package .
```

This creates a `.tgz` file (e.g., `velocis-1.0.0.tgz`) in the current directory.

Move it to your GitHub Pages directory (e.g., the root of your GitHub repo):

```bash
mv velocis-1.0.0.tgz ../../
cd ../../
```

### 2. Create or Update `index.yaml`

Run:

```bash
helm repo index . --url https://<username>.github.io/velocis-helm-charts
```

> Replace `<username>` with your GitHub username.

This creates (or updates) an `index.yaml` file pointing to your chart archive.

### 3. Commit and Push to GitHub

```bash
git add .
git commit -m "Add Helm chart velocis-1.0.0"
git push origin main
```

Make sure the `index.yaml` and `.tgz` file are pushed to GitHub and visible at your GitHub Pages URL.

---

## 🚀 Install Helm Chart

Once GitHub Pages is published:

### 1. Add the GitHub Helm repo locally

```bash
helm repo add velocis https://<username>.github.io/velocis-helm-charts
```

### 2. Update repositories

```bash
helm repo update
```

### 3. Dry run install

```bash
helm install test velocis/velocis --dry-run
```

### 4. Actual install

```bash
helm install test velocis/velocis
```

---

## 📌 Notes

* Update `Chart.yaml` with a new version number whenever you change chart content.
* Always regenerate `index.yaml` using `helm repo index . --url ...` after adding a new chart.
* GitHub Pages takes a few seconds to reflect changes after pushing.

---

## 📂 Resources

* Helm Docs: [https://helm.sh/docs](https://helm.sh/docs)
* GitHub Pages Docs: [https://pages.github.com/](https://pages.github.com/)
* Helm Chart Repositories: [https://helm.sh/docs/topics/chart\_repository/](https://helm.sh/docs/topics/chart_repository/)


