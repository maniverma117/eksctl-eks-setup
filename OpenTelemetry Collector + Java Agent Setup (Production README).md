# 📜 **OpenTelemetry Collector + Java Agent Setup (Production README)**

This setup enables **full observability (metrics, logs, traces)** using:

* OpenTelemetry Collector (central pipeline)
* OpenTelemetry Java Agent (app instrumentation)
* Prometheus (metrics)
* Grafana Loki (logs)
* Grafana Tempo (traces)

---

# 🧠 Architecture Overview

```text
Java App (OTel Agent)
        ↓ (OTLP)
OTel Collector (Deployment)
   ├── Metrics → Prometheus (scrape)
   ├── Logs    → Loki (push)
   └── Traces  → Tempo (push)
```

---

# 🚀 Prerequisites

* EKS cluster
* Prometheus, Loki, Tempo deployed
* Grafana configured
* Node labeled:

```bash
kubectl label node <node-name> role=monitoring
```

---

# 🛠️ PART 1 — OpenTelemetry Collector Setup

---

## 1. Add Helm Repo

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

---

## 2. Create `otel-values.yaml`

```bash
cat << EOF > otel-values.yaml

mode: deployment
replicaCount: 2

image:
  repository: otel/opentelemetry-collector-contrib

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

service:
  type: ClusterIP
  ports:
    - name: otlp-grpc
      port: 4317
    - name: otlp-http
      port: 4318
    - name: metrics
      port: 8889

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: role
              operator: In
              values:
                - monitoring

config:
  receivers:
    otlp:
      protocols:
        grpc:
        http:

  processors:
    memory_limiter:
      limit_mib: 512
      spike_limit_mib: 128
      check_interval: 5s
    batch:

  exporters:
    prometheus:
      endpoint: "0.0.0.0:8889"

    loki:
      endpoint: http://loki:3100/loki/api/v1/push

    otlp/tempo:
      endpoint: tempo:4317
      tls:
        insecure: true

  service:
    pipelines:
      metrics:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [prometheus]

      logs:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [loki]

      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp/tempo]

EOF
```

---

## 3. Deploy Collector

```bash
helm upgrade --install otel-collector \
  -f otel-values.yaml \
  open-telemetry/opentelemetry-collector \
  -n monitoring --create-namespace
```

---

# 🔗 PART 2 — Prometheus Integration

👉 Prometheus must scrape OTel

Create ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    release: prometheus
  name: otel-collector
  namespace: monitoring
spec:
  endpoints:
  - interval: 15s
    port: metrics
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      app.kubernetes.io/instance: otel-collector
      app.kubernetes.io/name: opentelemetry-collector
      component: standalone-collector
```

---

# 🛠️ PART 3 — Java Agent Setup (CRITICAL 🔥)

---

## 1. Download Agent

```bash
wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
```

---

## 2. Run Application with OTel

```bash
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.service.name=my-java-app \
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
  -Dotel.metrics.exporter=otlp \
  -Dotel.traces.exporter=otlp \
  -Dotel.logs.exporter=otlp \
  -Dotel.instrumentation.logback-appender.enabled=true \
  -jar app.jar
```

---

# 🔥 What This Enables

| Feature     | Status |
| ----------- | ------ |
| Metrics     | ✅      |
| Logs        | ✅      |
| Traces      | ✅      |
| Correlation | ✅      |

---

# 🔗 Correlation (Logs ↔ Traces)

👉 Logs will contain:

```json
{
  "trace_id": "abc123",
  "span_id": "xyz789"
}
```

👉 In Grafana:

* Open trace → click → view logs
* Logs automatically filtered by `trace_id`

---

# 📊 Grafana Setup

In Grafana:

### Add Data Sources:

* Prometheus
* Loki
* Tempo

### Enable:

* Trace to logs
* Logs to trace

---

# 🚨 Disable Promtail (Important)

```bash
helm upgrade loki grafana/loki \
  --set promtail.enabled=false
```

👉 Or remove promtail DaemonSet:

```bash
kubectl delete ds promtail -n monitoring
```

---

# 🔍 Verification Steps

---

## 1. Check Collector Logs

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=opentelemetry-collector
```

---

## 2. Check Metrics

```bash
kubectl port-forward svc/otel-collector 8889:8889 -n monitoring
curl localhost:8889/metrics
```

---

## 3. Check Logs (Loki)

```logql
{service="my-java-app"}
```

---

## 4. Check Traces (Tempo)

Search:

```text
service.name="my-java-app"
```

---

# ⚠️ Common Mistakes

---

❌ Forgetting `logs.exporter=otlp`
❌ Not enabling logback appender
❌ Missing ServiceMonitor
❌ Wrong Loki endpoint

---

# 🧠 Final Architecture

```text
App → OTel Collector → Prometheus
                     → Loki
                     → Tempo
```

---

# 🔥 Summary

This setup provides:

* Unified telemetry pipeline
* Automatic trace ↔ log correlation
* Reduced complexity (no Promtail)
* Production-ready observability

