# 🚀 Spring Boot Observability with Prometheus, Alertmanager & Slack

This repository provides a **complete observability setup** for Spring Boot applications using:

* 📊 Prometheus (metrics)
* 🚨 Prometheus Rules (alerting)
* 🔔 Alertmanager (routing)
* 💬 Slack (notifications)
* 📈 Micrometer (Spring Boot metrics)

---

# 📌 Architecture

```
Spring Boot (Micrometer)
        ↓
/actuator/prometheus
        ↓
Prometheus
        ↓
PrometheusRule (alerts)
        ↓
Alertmanager
        ↓
Slack
```

---

# 📊 Spring Boot Metrics Setup

Add dependency:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Enable actuator:

```properties
management.endpoints.web.exposure.include=*
management.endpoint.prometheus.enabled=true
```

---

# 🚨 PrometheusRule.yaml (FULL CONFIG)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: springboot-observability-rules
  namespace: monitoring
  labels:
    app: kube-prometheus-stack
    release: prometheus
spec:
  groups:
  - name: springboot-observability
    interval: 30s
    rules:

    - alert: High5xxErrorRate
      expr: |
        (
          sum(rate(http_server_requests_seconds_count{status=~"5..", uri!="/actuator/prometheus"}[5m]))
          /
          sum(rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m]))
        ) > 0.05
      for: 2m
      labels:
        severity: critical
        service: springboot
      annotations:
        summary: "High 5xx error rate"
        description: "More than 5% requests failing with 5xx errors"

    - alert: LowSuccessRate
      expr: |
        (
          sum(rate(http_server_requests_seconds_count{status=~"2..", uri!="/actuator/prometheus"}[5m]))
          /
          sum(rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m]))
        ) < 0.9
      for: 2m
      labels:
        severity: warning
        service: springboot
      annotations:
        summary: "Low success rate"
        description: "Less than 90% requests are successful"

    - alert: HighLatencyP95
      expr: |
        histogram_quantile(0.95,
          sum(rate(http_server_requests_seconds_bucket{uri!="/actuator/prometheus"}[5m])) by (le)
        ) > 1
      for: 2m
      labels:
        severity: warning
        service: springboot
      annotations:
        summary: "High latency P95"
        description: "P95 latency is greater than 1 second"

    - alert: HighLatencyP99
      expr: |
        histogram_quantile(0.99,
          sum(rate(http_server_requests_seconds_bucket{uri!="/actuator/prometheus"}[5m])) by (le)
        ) > 2
      for: 2m
      labels:
        severity: critical
        service: springboot
      annotations:
        summary: "High latency P99"
        description: "P99 latency is greater than 2 seconds"

    - alert: HighExceptions
      expr: |
        sum(rate(http_server_requests_seconds_count{outcome="SERVER_ERROR"}[5m])) > 5
      for: 2m
      labels:
        severity: critical
        service: springboot
      annotations:
        summary: "High exceptions detected"
        description: "Too many server errors in last 5 minutes"

    - alert: HighTrafficSpike
      expr: |
        sum(rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[1m])) > 100
      for: 1m
      labels:
        severity: info
        service: springboot
      annotations:
        summary: "Traffic spike detected"
        description: "Request rate unusually high"

    - alert: NoTraffic
      expr: |
        sum(rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m])) == 0
      for: 3m
      labels:
        severity: critical
        service: springboot
      annotations:
        summary: "No traffic detected"
        description: "Application might be down"

    - alert: HighLatencyPerURI
      expr: |
        histogram_quantile(0.95,
          sum(rate(http_server_requests_seconds_bucket{uri!="/actuator/prometheus"}[5m])) by (uri, le)
        ) > 1.5
      for: 3m
      labels:
        severity: warning
        service: springboot
      annotations:
        summary: "High latency on specific URI"
        description: "Endpoint {{ $labels.uri }} has high latency"

    - alert: High5xxPerURI
      expr: |
        sum by(uri)(
          rate(http_server_requests_seconds_count{status=~"5..", uri!="/actuator/prometheus"}[5m])
        ) > 1
      for: 2m
      labels:
        severity: warning
        service: springboot
      annotations:
        summary: "High errors on endpoint"
        description: "Endpoint {{ $labels.uri }} returning high errors"
```

---

# 🔔 Alertmanager.yaml (FULL CONFIG)

```yaml
global:
  resolve_timeout: 5m

inhibit_rules:
- source_matchers:
    - severity = critical
  target_matchers:
    - severity =~ warning|info
  equal: [namespace, alertname]

- source_matchers:
    - severity = warning
  target_matchers:
    - severity = info
  equal: [namespace, alertname]

- source_matchers:
    - alertname = InfoInhibitor
  target_matchers:
    - severity = info
  equal: [namespace]

- target_matchers:
    - alertname = InfoInhibitor

route:
  group_by: [alertname, namespace]
  group_wait: 10s
  group_interval: 2m
  repeat_interval: 1h
  receiver: "slack-alerts"

  routes:
  - receiver: slack-alerts
    matchers:
      - alertname=~"High5xxErrorRate|LowSuccessRate|HighLatencyP95|HighLatencyP99|HighExceptions|HighTrafficSpike|NoTraffic|HighLatencyPerURI|High5xxPerURI"

receivers:
- name: "slack-alerts"
  slack_configs:
  - api_url: "<YOUR_SLACK_WEBHOOK>"
    channel: "#general"
    send_resolved: true
    username: "Alertmanager"
    icon_emoji: ":rotating_light:"
    title: '🚨 {{ .CommonLabels.alertname }}'
    color: '{{ if eq .CommonLabels.severity "critical" }}danger{{ else if eq .CommonLabels.severity "warning" }}warning{{ else }}good{{ end }}'
    text: |
      *Severity:* {{ .CommonLabels.severity }}
      *Service:* {{ .CommonLabels.service }}
      *Namespace:* {{ .CommonLabels.namespace }}
      *Description:* {{ .CommonAnnotations.description }}

templates:
- /etc/alertmanager/config/*.tmpl
```

---

# 🚀 Deployment

```bash
kubectl apply -f PrometheusRule.yaml
kubectl apply -f alertmanager.yaml
```

Restart:

```bash
kubectl rollout restart deployment alertmanager-prometheus-kube-prometheus-alertmanager -n monitoring
```

---

# 🧪 Testing

```bash
curl http://<app-url>/error
```

---

# ✅ Output

* Alerts visible in Prometheus
* Routed via Alertmanager
* Delivered to Slack

---

# 🔐 Security

⚠️ Never expose:

* Slack webhook
* AWS SMTP credentials

Use:

* Kubernetes Secrets
* AWS Secrets Manager

---

# 📈 Outcome

* Golden signals monitoring ✅
* Real-time alerts ✅
* Endpoint-level visibility ✅
* Production-ready observability ✅

---

🔥 This is a **complete SRE-grade alerting system**
