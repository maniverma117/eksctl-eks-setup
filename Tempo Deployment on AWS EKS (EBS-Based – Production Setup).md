# 📜 **Tempo Deployment on AWS EKS (EBS-Based – Production Setup)**

This section describes how to deploy Grafana Tempo on AWS EKS using:

* Amazon EBS for trace storage
* Default Helm chart configuration
* Custom node affinity (dedicated monitoring node)
* OpenTelemetry for trace ingestion

---

# 🚀 Prerequisites

* AWS EKS Cluster
* `kubectl` configured
* Helm 3 installed
* Node labeled:

```bash
kubectl label node <node-name> role=monitoring
```

* StorageClass available:

```bash
kubectl get storageclass
```

Expected:

```text
ebs-sc
```

---

# 🧠 Approach (Important)

👉 Instead of overriding everything, we:

* ✅ Keep **default Tempo config**
* ✅ Override only:

  * Affinity
  * Persistence

👉 This is the **best practice** (minimal override, stable upgrades)

---

# 🛠️ Setup Instructions

---

## 1. Add Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## 2. Pull Chart (Optional)

```bash
helm pull grafana/tempo --untar
```

---

## 3. Create Custom Values File

Create `tempo-values.yaml`:

```bash
cat << EOF > tempo-values.yaml

# --- Persistence (EBS Storage) ---
persistence:
  enabled: true
  enableStatefulSetAutoDeletePVC: false
  storageClassName: ebs-sc
  accessModes:
    - ReadWriteOnce
  size: 50Gi

# --- Node Affinity (Dedicated Monitoring Node) ---
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: role
              operator: In
              values:
                - monitoring

EOF
```

---

## 4. Dry Run

```bash
helm upgrade --install tempo \
  -f tempo-values.yaml \
  grafana/tempo \
  -n monitoring --create-namespace \
  --dry-run
```

---

## 5. Deploy Tempo

```bash
helm upgrade --install tempo \
  -f tempo-values.yaml \
  grafana/tempo \
  -n monitoring --create-namespace
```

---

# ✅ Post-Deployment Verification

---

## Check Pods

```bash
kubectl get pods -n monitoring -o wide
```

👉 Ensure:

* Pod is running
* Scheduled on `monitoring` node

---

## Check PVC

```bash
kubectl get pvc -n monitoring
```

👉 Expected:

* 50Gi PVC bound
* Using `ebs-sc`

---

## Port Forward

```bash
kubectl port-forward svc/tempo 3200:3200 -n monitoring
```

Test:

```bash
curl http://localhost:3200/ready
```

---

# 🔗 OpenTelemetry Integration

In OpenTelemetry Collector:

```yaml
exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
```

---

# 📊 Grafana Integration

In Grafana:

* Data Source Type: Tempo
* URL:

```text
http://tempo.monitoring.svc.cluster.local:3200
```

---

# 🔍 Testing Traces

Go to Grafana → Explore → Tempo

Query:

```text
service.name="your-app"
```

---

# 🧠 What Default Config Already Provides

From Helm defaults :

### ✅ Receivers enabled

* OTLP (4317, 4318)
* Jaeger
* OpenCensus

👉 Your OTel collector can send traces directly

---

### ✅ Storage (local filesystem)

```yaml
backend: local
path: /var/tempo/traces
```

👉 This is stored on your EBS volume

---

### ✅ WAL (Write Ahead Log)

```yaml
wal:
  path: /var/tempo/wal
```

👉 Ensures:

* crash safety
* data durability

---

### ✅ Retention

```yaml
retention: 24h
```

👉 Default = 1 day

---

# ⚠️ Recommended Improvement (Important)

Increase retention:

```yaml
tempo:
  retention: 168h   # 7 days
```

---

# 🧠 Architecture Overview

```text
Java App → OTel → Tempo → EBS
                              ↓
                          Grafana
```

---

# ⚠️ Limitations (EBS-Based Tempo)

| Area    | Impact            |
| ------- | ----------------- |
| Scaling | Single node       |
| HA      | No replication    |
| Storage | Disk bound        |
| Failure | Node failure risk |

---

# 🚨 Recommended Alerts

---

## Disk Usage

```promql
(node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.2
```

---

## Tempo Errors

```logql
{job="tempo"} |= "error"
```

---

# 🔥 Best Practices

---

## 1. Sampling (VERY IMPORTANT)

Control trace volume:

```yaml
processors:
  probabilistic_sampler:
    sampling_percentage: 10
```

---

## 2. Service Naming

Ensure:

```text
service.name=my-service
env=prod
```

---

## 3. Avoid Huge Traces

* Large payloads slow queries
* Increase storage usage

---

# 📎 Notes

---

## Storage Monitoring

```bash
kubectl exec -it <tempo-pod> -- df -h
```

---

## Restart Behavior

* Data persists (EBS)
* WAL ensures safety

---

# 🧠 Summary

This setup provides:

* Distributed tracing using Tempo
* Persistent storage on EBS
* Integration with OpenTelemetry
* Visualization in Grafana

---

# 🚀 Next Steps

You now have:

* Prometheus ✅
* Loki ✅
* Tempo ✅

👉 Next logical step:

* OpenTelemetry Collector (MOST IMPORTANT)
