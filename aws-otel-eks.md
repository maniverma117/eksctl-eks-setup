# AWS Observability on EKS (CloudWatch & ADOT)

> **Production Standard Operating Procedure (SOP)**
> Observability on Amazon EKS using AWS-managed CloudWatch components and user-managed ADOT for tracing.
>
> Source: 

---

## 1. Purpose

This document defines a **production-ready and AWS-supported** approach to enable observability on **Amazon EKS** using:

* **CloudWatch Container Insights** – Metrics
* **Fluent Bit** – Logs
* **AWS Distro for OpenTelemetry (ADOT)** – Traces

This design avoids CRD conflicts, ownership issues, and unsupported configurations.

---

## 2. Scope

* Amazon EKS clusters
* Kubernetes version **≥ 1.24**
* AWS-managed **CloudWatch Observability add-on**
* User-managed **ADOT Collector (Tracing only)**

---

## 3. Architecture Overview

```
Application Pods
├── Metrics ──> CloudWatch Agent ──> CloudWatch (Container Insights)
├── Logs ─────> Fluent Bit ─────────> CloudWatch Logs
└── Traces ───> ADOT Collector ─────> AWS X-Ray
```

### Namespace Ownership (MANDATORY)

| Component                | Namespace         | Ownership    |
| ------------------------ | ----------------- | ------------ |
| CloudWatch Agent         | amazon-cloudwatch | AWS-managed  |
| Fluent Bit               | amazon-cloudwatch | AWS-managed  |
| Observability Controller | amazon-cloudwatch | AWS-managed  |
| ADOT Collector (Tracing) | aws-otel          | User-managed |

⚠️ **Never deploy custom resources inside `amazon-cloudwatch`**

---

## 4. Prerequisites

### 4.1 Tools

* `aws` CLI
* `kubectl`
* `eksctl`
* `helm`

### 4.2 Permissions

* IAM role creation
* IRSA (IAM Roles for Service Accounts)
* EKS add-on management permissions

---

## 5. Environment Variables

```bash
export AWS_REGION="ap-south-1"
export EKS_CLUSTER_NAME="<cluster-name>"

# AWS-managed namespace
export OBS_NAMESPACE="amazon-cloudwatch"

# User-managed ADOT namespace
export ADOT_NAMESPACE="aws-otel"
export ADOT_SA_NAME="adot-collector"
```

---

## 6. Enable CloudWatch Observability Add-on

### 6.1 Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster $EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --approve
```

---

### 6.2 Create Namespace (idempotent)

```bash
kubectl create namespace $OBS_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

### 6.3 Create IRSA for CloudWatch

```bash
eksctl create iamserviceaccount \
  --cluster $EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --namespace $OBS_NAMESPACE \
  --name aws-observability \
  --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --attach-policy-arn arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess \
  --approve
```

---

### 6.4 Install CloudWatch Observability Add-on

```bash
aws eks create-addon \
  --cluster-name $EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --addon-name amazon-cloudwatch-observability \
  --resolve-conflicts OVERWRITE
```

---

### 6.5 Verification

```bash
kubectl get pods -n $OBS_NAMESPACE
```

Expected:

* cloudwatch-agent
* fluent-bit
* observability controller

---

## 7. Enable ADOT Tracing (User-Managed)

### 7.1 Create ADOT Namespace

```bash
kubectl create namespace $ADOT_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

### 7.2 Create IRSA for ADOT Collector

```bash
eksctl create iamserviceaccount \
  --cluster $EKS_CLUSTER_NAME \
  --region $AWS_REGION \
  --namespace $ADOT_NAMESPACE \
  --name $ADOT_SA_NAME \
  --attach-policy-arn arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess \
  --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --approve
```

---

### 7.3 Install ADOT Collector (Tracing Only)

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm upgrade --install adot-collector open-telemetry/opentelemetry-collector \
  --namespace $ADOT_NAMESPACE \
  --set mode=deployment \
  --set image.repository=public.ecr.aws/aws-observability/aws-otel-collector \
  --set image.tag=latest \
  --set serviceAccount.create=false \
  --set serviceAccount.name=$ADOT_SA_NAME \
  --set config.receivers.otlp.protocols.grpc.endpoint=0.0.0.0:4317 \
  --set config.receivers.otlp.protocols.http.endpoint=0.0.0.0:4318 \
  --set config.exporters.awsxray.region=$AWS_REGION \
  --set config.service.pipelines.traces.receivers={otlp} \
  --set config.service.pipelines.traces.exporters={awsxray}
```

---

## 8. Application Instrumentation

### Common (All Languages)

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://adot-collector.aws-otel.svc.cluster.local:4317"
export OTEL_TRACES_EXPORTER="otlp"
export OTEL_SERVICE_NAME="<service-name>"
export OTEL_METRICS_EXPORTER="none"
export OTEL_LOGS_EXPORTER="none"
export OTEL_TRACES_EXPORTER="otlp"
export OTEL_EXPORTER_OTLP_PROTOCOL="grpc"
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=<environment_name>,service.name=<service-name>"
export OTEL_PROPAGATORS="tracecontext,baggage"
```

---

### Java

```bash
-javaagent:/otel/opentelemetry-javaagent.jar
```

---

### Node.js

```bash
export NODE_OPTIONS="--require @opentelemetry/auto-instrumentations-node/register"
```

---

## 9. Validation Checklist

* ✅ Metrics visible in **CloudWatch → Container Insights**
* ✅ Logs visible in **CloudWatch Logs**
* ✅ Traces visible in **AWS X-Ray**
* ✅ No `opentelemetrycollector` resources in `amazon-cloudwatch`
* ✅ ADOT only deployed in `aws-otel`

---

## 10. Operational Guidelines

### ✅ Do

* Use IRSA everywhere
* Keep AWS-managed components isolated
* Manage ADOT via Helm
* Use OTLP from applications

### ❌ Don’t

* Modify resources in `amazon-cloudwatch`
* Deploy ADOT in AWS-managed namespaces
* Install multiple OTel operators
* Use DaemonSet for ADOT unless explicitly required

---

## 11. Ownership & Review

* **Owner:** DevOps / Platform Team
* **Review Cycle:** Every 6 months or after AWS add-on updates
