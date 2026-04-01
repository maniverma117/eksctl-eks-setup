# 📜 **Loki Deployment on AWS EKS (EBS-Based Production Setup)**

This repository contains setup instructions and Helm configuration to deploy Grafana Loki on an AWS EKS cluster using:

* Amazon EBS for log storage
* Promtail for log collection
* Dedicated monitoring nodes
* Retention using compactor

---

# 🚀 Prerequisites

* AWS EKS Cluster
* `kubectl` configured
* Helm 3 installed
* Node labeled with:

```bash
role=Prd-Monitoring-NG-1
```

* StorageClass:

```bash
ebs-sc
```

Verify:

```bash
kubectl get storageclass
```

---

# 🛠️ Setup Instructions

---

## 1. Add Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## 2. Pull and Untar the Chart

```bash
helm pull grafana/loki --untar
```

---

## 3. Create Custom Values File

Create `loki-values.yaml`:

```bash
cat << EOF > loki-values.yaml
loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: boltdb-shipper
        object_store: filesystem
        schema: v12
        index:
          prefix: loki_index_
          period: 24h

  storage_config:
    boltdb_shipper:
      active_index_directory: /var/loki/index
      cache_location: /var/loki/cache
      shared_store: filesystem

    filesystem:
      directory: /var/loki/chunks

  compactor:
    working_directory: /var/loki/compactor
    retention_enabled: true

  limits_config:
    retention_period: 672h

  persistence:
    enabled: true
    storageClassName: ebs-sc
    accessModes:
      - ReadWriteOnce
    size: 250Gi

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values:
                  - Prd-Monitoring-NG-1

promtail:
  enabled: true
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values:
                  - Prd-Monitoring-NG-1
EOF
```

---

## 4. Dry Run

```bash
helm upgrade --install loki \
  -f loki-values.yaml \
  loki/ \
  -n monitoring --create-namespace \
  --dry-run
```

---

## 5. Deploy Loki

```bash
helm upgrade --install loki \
  -f loki-values.yaml \
  loki/ \
  -n monitoring --create-namespace
```

---

# ✅ Post-Deployment Verification

---

## Check Pods

```bash
kubectl get pods -n monitoring -o wide
```

---

## Check PVC

```bash
kubectl get pvc -n monitoring
```

---

## Check Logs Ingestion

```bash
kubectl logs -n monitoring -l app=loki
```

---

## Port Forward Loki

```bash
kubectl port-forward svc/loki 3100:3100 -n monitoring
```

Test:

```bash
curl http://localhost:3100/ready
```

---

# 📊 Integrate with Grafana

In Grafana:

### Add Data Source:

* Type: Loki
* URL:

```text
http://loki.monitoring.svc.cluster.local:3100
```

---

# 🔍 Test Query

```logql
{namespace="default"}
```

---

# 🧠 Architecture Overview

```text
Kubernetes Logs → Promtail → Loki → EBS
                                 ↓
                             Grafana
```

---

# ⚠️ Important Notes

---

## 1. No `table_manager`

* Deprecated
* Replaced by compactor

---

## 2. EBS Limitations

* Single-node scaling
* Disk-bound performance
* Monitor disk usage carefully

---

## 3. Retention

Configured:

```yaml
retention_period: 672h (28 days)
```

Handled by:

* Loki compactor

---

# 🚨 Recommended Alerts

---

## Disk Usage

```promql
(node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.2
```

---

## Loki Errors

```logql
{job="loki"} |= "error"
```

---

# 🔥 Best Practices

---

## Label Strategy

Use:

```text
service=my-app
env=prod
namespace=default
```

Avoid:

```text
user_id=12345
request_id=random
```

---

## Drop Noisy Logs

Promtail config:

```yaml
pipeline_stages:
  - drop:
      expression: ".*healthcheck.*"
```

---

## Storage Recommendations

* Use **gp3 EBS**
* Enable high IOPS if needed

---

# 📎 Notes

---

## Node Label

Apply before deployment:

```bash
kubectl label node <node-name> role=Prd-Monitoring-NG-1
```

---

## StorageClass Check

```bash
kubectl get storageclass
```

---

# 🧠 Summary

This setup provides:

* Log aggregation using Loki
* Persistent storage on EBS
* Log collection via Promtail
* Retention using compactor
* Integration with Grafana

---

# 🚀 Next Steps

You can extend this setup with:

* Grafana Tempo → tracing
* OpenTelemetry Collector → unified pipeline
* Prometheus → metrics

---

If you want, next I can create:

👉 Tempo README (same level)
👉 OTel Collector README (with pipelines)
👉 Full “observability stack repo structure” (very useful for your project)
