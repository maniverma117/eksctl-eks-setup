# 📜 Promtail Deployment on AWS EKS (Production Setup with Loki)

This repository contains setup instructions and Helm configuration to deploy **Promtail** on an AWS EKS cluster for log collection and forwarding to **Grafana Loki**.

Promtail runs as a **DaemonSet** and collects logs from all nodes, enriches them with labels, and pushes them to Loki.

---

## 🚀 Prerequisites

* AWS EKS Cluster
* `kubectl` configured
* Helm 3 installed
* Loki already deployed and reachable:

  ```
  http://loki.monitoring.svc.cluster.local:3100
  ```
* Nodes accessible with log paths:

  ```
  /var/log
  /var/lib/docker/containers
  ```

---

## 🛠️ Setup Instructions

### 1. Add Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

### 2. Pull and Untar the Chart

```bash
helm pull grafana/promtail --untar
```

---

### 3. Create Custom Values File

Create `promtail-values.yaml`:

```bash
cat << EOF > promtail-values.yaml
config:
  clients:
    - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push

EOF
```

---

### 4. Dry Run

```bash
helm upgrade --install promtail \
  -f promtail-values.yaml \
  promtail/ \
  -n monitoring --create-namespace \
  --dry-run
```

---

### 5. Deploy Promtail

```bash
helm upgrade --install promtail \
  -f promtail-values.yaml \
  promtail/ \
  -n monitoring --create-namespace
```

---

## ✅ Post-Deployment Verification

### Check Pods

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail -o wide
```

---

### Check Logs

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=promtail
```

---

### Verify Loki Ingestion

In Loki:

```bash
{namespace="default"}
```

---

## 🧠 Architecture Overview

```
Kubernetes Logs → Promtail (DaemonSet) → Loki → EBS
                                              ↓
                                          Grafana
```

---

## ⚠️ Important Notes

### 1. Positions File

* Stored at:

  ```
  /run/promtail/positions.yaml
  ```
* Prevents duplicate log ingestion after restart

---

### 2. Log Sources

Promtail reads from:

* `/var/log/pods`
* `/var/log/containers`
* `/var/lib/docker/containers`

---

### 3. Label Strategy (VERY IMPORTANT 🔥)

Use controlled labels:

```
namespace
pod
container
node
```

Avoid high cardinality labels like:

```
request_id
user_id
session_id
```

---

### 4. Scaling Consideration

For your case (🔥 200 microservices, 400–500 pods):

* DaemonSet = **Perfect choice**
* One Promtail per node → efficient
* No need for sidecars (reduces cost + complexity)

---

## 🚨 Recommended Alerts

### Promtail Errors

```
{job="promtail"} |= "error"
```

---

### Log Ingestion Rate Drop

Alert if logs suddenly drop → indicates pipeline issue

---

## 🔥 Best Practices

### 1. Drop Noisy Logs

```yaml
pipeline_stages:
  - drop:
      expression: ".*healthcheck.*"
```

---

### 2. Use Structured Logs (JSON)

Promtail parses JSON automatically → better querying in Loki

---

### 3. Avoid Overloading Loki

* Control log volume
* Use sampling if needed

---

### 4. Resource Optimization

* Tune CPU/memory based on node size
* Avoid over-provisioning

---

## 📎 Notes

### Node Label (Optional)

```bash
kubectl label node <node-name> role=monitoring
```

---

## 🧠 Summary

This setup provides:

* Centralized log collection using Promtail
* Efficient node-level scraping (DaemonSet)
* Seamless integration with Loki
* Scalable for large microservices architecture
* Production-safe configuration

---

## 🚀 Next Steps

You can extend this setup with:

* **Grafana Tempo** → Distributed tracing
* **OpenTelemetry Collector** → Logs + Metrics + Traces unified pipeline
* **Prometheus + Micrometer** → Metrics

---

If you want, next step I can help you:

👉 Correlate **logs + traces (trace_id / span_id)** between Promtail + Tempo (this is where most teams struggle 👍)
