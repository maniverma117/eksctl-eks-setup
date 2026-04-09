# 🚨 Prometheus + Alertmanager Slack Integration (Complete Guide)

This guide helps you:
- Send alerts from Prometheus → Alertmanager → Slack
- Fix common issues (missing labels, grouping problems)
- Use production-ready alert rules (Golden Signals)

---

# 🚀 Step 1: Create Slack Webhook

1. Go to https://api.slack.com/apps  
2. Click **Create New App → From scratch**
3. Enable **Incoming Webhooks**
4. Click **Add New Webhook**
5. Select channel (e.g. `#alerts`)
6. Copy webhook URL

Example:
```

[https://hooks.slack.com/services/XXXX/XXXX/XXXX](https://hooks.slack.com/services/XXXX/XXXX/XXXX)

````

---

# ⚙️ Step 2: Configure Alertmanager

Update `alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m

# =========================
# 🚫 INHIBIT RULES
# =========================
inhibit_rules:
- source_matchers:
    - severity="critical"
  target_matchers:
    - severity=~"warning|info"
  equal: [namespace, alertname, application]

- source_matchers:
    - severity="warning"
  target_matchers:
    - severity="info"
  equal: [namespace, alertname, application]

- source_matchers:
    - alertname="InfoInhibitor"
  target_matchers:
    - severity="info"
  equal: [namespace]

- target_matchers:
    - alertname="InfoInhibitor"

# =========================
# 🚦 ROUTING
# =========================
route:
  receiver: "slack-alerts"

  # 🔥 CRITICAL FIX (no more empty service)
  group_by: [alertname, application, service, namespace]

  group_wait: 10s
  group_interval: 2m
  repeat_interval: 1h

  routes:

  # 🚨 Spring Boot Alerts
  - receiver: slack-alerts
    matchers:
      - alertname=~"High5xxErrorRate|High4xxErrorRate|LowSuccessRate|HighLatencyP95|HighLatencyP99|HighTrafficSpike|NoTraffic|HighLatencyPerURI|High5xxPerURI|High4xxPerURI|High5xxErrorRatioPerURI|High4xxErrorRatioPerURI"

  # 🚨 K8s Alerts
  - receiver: slack-alerts
    matchers:
      - alertname=~"KubePodCrashLooping|KubePodNotReady|KubeDeploymentGenerationMismatch|KubeDeploymentReplicasMismatch|KubeDeploymentRolloutStuck|KubeStatefulSetReplicasMismatch|KubeStatefulSetGenerationMismatch|KubeStatefulSetUpdateNotRolledOut|KubeDaemonSetRolloutStuck|KubeContainerWaiting|KubeDaemonSetNotScheduled|KubeDaemonSetMisScheduled|KubeJobNotCompleted|KubeJobFailed|KubeHpaReplicasMismatch|KubeHpaMaxedOut"

# =========================
# 📩 RECEIVER (SLACK)
# =========================
receivers:
- name: "slack-alerts"
  slack_configs:
  - api_url: "https://hooks.slack.com/services/T025JRLQD2xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    channel: "#general"
    send_resolved: true
    username: "Alertmanager"
    icon_emoji: ":rotating_light:"

    title: '🚨 {{ .CommonLabels.alertname }}'

    color: '{{ if eq .CommonLabels.severity "critical" }}danger{{ else if eq .CommonLabels.severity "warning" }}warning{{ else }}good{{ end }}'

    # 🔥 FIX: Use range to avoid empty labels
    text: |
      *Severity:* {{ .CommonLabels.severity }}
      {{ range .Alerts }}
      *Service:* {{ .Labels.service }}
      *Application:* {{ .Labels.application }}
      *Namespace:* {{ .Labels.namespace }}
      *Description:* {{ .Annotations.description }}
      ---
      {{ end }}

templates:
- /etc/alertmanager/config/*.tmpl
````

---

# 📊 Step 3: Add Prometheus Alert Rules

# Apply the following `vim PrometheusRule.yaml`:

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

    # =========================
    # 🚨 ERROR RATE (5xx - GLOBAL)
    # =========================
    - alert: High5xxErrorRate
      expr: |
        (
          sum by(application)(
            rate(http_server_requests_seconds_count{status=~"5..", uri!="/actuator/prometheus"}[5m])
          )
          /
          sum by(application)(
            rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m])
          )
        ) > 0.05
      for: 2m
      labels:
        severity: critical
        service: "{{ $labels.application }}"
      annotations:
        summary: "High 5xx error rate"
        description: "Service {{ $labels.application }} has >5% 5xx errors"

    # =========================
    # 🚨 ERROR RATE (4xx - GLOBAL)
    # =========================
    - alert: High4xxErrorRate
      expr: |
        (
          sum by(application)(
            rate(http_server_requests_seconds_count{status=~"4..", uri!="/actuator/prometheus"}[5m])
          )
          /
          sum by(application)(
            rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m])
          )
        ) > 0.1
      for: 2m
      labels:
        severity: warning
        service: "{{ $labels.application }}"
      annotations:
        summary: "High 4xx error rate"
        description: "Service {{ $labels.application }} has >10% client errors"

    # =========================
    # 🚨 LOW SUCCESS RATE
    # =========================
    - alert: LowSuccessRate
      expr: |
        (
          sum by(application)(
            rate(http_server_requests_seconds_count{status=~"2..", uri!="/actuator/prometheus"}[5m])
          )
          /
          sum by(application)(
            rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m])
          )
        ) < 0.9
      for: 2m
      labels:
        severity: warning
        service: "{{ $labels.application }}"
      annotations:
        summary: "Low success rate"
        description: "Service {{ $labels.application }} has <90% success rate"

    # =========================
    # 🚨 LATENCY P95
    # =========================
    - alert: HighLatencyP95
      expr: |
        histogram_quantile(0.95,
          sum by(application, le)(
            rate(http_server_requests_seconds_bucket{uri!="/actuator/prometheus"}[5m])
          )
        ) > 1
      for: 2m
      labels:
        severity: warning
        service: "{{ $labels.application }}"
      annotations:
        summary: "High latency P95"
        description: "Service {{ $labels.application }} P95 latency > 1s"

    # =========================
    # 🚨 LATENCY P99
    # =========================
    - alert: HighLatencyP99
      expr: |
        histogram_quantile(0.99,
          sum by(application, le)(
            rate(http_server_requests_seconds_bucket{uri!="/actuator/prometheus"}[5m])
          )
        ) > 2
      for: 2m
      labels:
        severity: critical
        service: "{{ $labels.application }}"
      annotations:
        summary: "High latency P99"
        description: "Service {{ $labels.application }} P99 latency > 2s"

    # =========================
    # 🚨 TRAFFIC SPIKE
    # =========================
    - alert: HighTrafficSpike
      expr: |
        sum by(application)(
          rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[1m])
        ) > 100
      for: 1m
      labels:
        severity: info
        service: "{{ $labels.application }}"
      annotations:
        summary: "Traffic spike detected"
        description: "Service {{ $labels.application }} traffic is unusually high"

    # =========================
    # 🚨 NO TRAFFIC
    # =========================
    - alert: NoTraffic
      expr: |
        sum by(application)(
          rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m])
        ) == 0
      for: 3m
      labels:
        severity: critical
        service: "{{ $labels.application }}"
      annotations:
        summary: "No traffic detected"
        description: "Service {{ $labels.application }} might be down"

    # =========================
    # 🚨 5xx PER URI (ABSOLUTE)
    # =========================
    - alert: High5xxPerURI
      expr: |
        sum by(application, uri)(
          rate(http_server_requests_seconds_count{status=~"5..", uri!="/actuator/prometheus"}[5m])
        ) > 1
      for: 2m
      labels:
        severity: critical
        service: "{{ $labels.application }}"
      annotations:
        summary: "High 5xx errors on endpoint"
        description: "Service {{ $labels.application }} - Endpoint {{ $labels.uri }} has server errors"

    # =========================
    # 🚨 4xx PER URI (ABSOLUTE)
    # =========================
    - alert: High4xxPerURI
      expr: |
        sum by(application, uri)(
          rate(http_server_requests_seconds_count{status=~"4..", uri!="/actuator/prometheus"}[5m])
        ) > 2
      for: 2m
      labels:
        severity: warning
        service: "{{ $labels.application }}"
      annotations:
        summary: "High 4xx errors on endpoint"
        description: "Service {{ $labels.application }} - Endpoint {{ $labels.uri }} has client errors"

    # =========================
    # 🚨 5xx PER URI (RATIO)
    # =========================
    - alert: High5xxErrorRatioPerURI
      expr: |
        (
          sum by(application, uri)(
            rate(http_server_requests_seconds_count{status=~"5..", uri!="/actuator/prometheus"}[5m])
          )
          /
          sum by(application, uri)(
            rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m])
          )
        ) > 0.05
      for: 2m
      labels:
        severity: critical
        service: "{{ $labels.application }}"
      annotations:
        summary: "High 5xx error ratio on endpoint"
        description: "Service {{ $labels.application }} - Endpoint {{ $labels.uri }} has >5% server errors"

    # =========================
    # 🚨 4xx PER URI (RATIO)
    # =========================
    - alert: High4xxErrorRatioPerURI
      expr: |
        (
          sum by(application, uri)(
            rate(http_server_requests_seconds_count{status=~"4..", uri!="/actuator/prometheus"}[5m])
          )
          /
          sum by(application, uri)(
            rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m])
          )
        ) > 0.1
      for: 2m
      labels:
        severity: warning
        service: "{{ $labels.application }}"
      annotations:
        summary: "High 4xx error ratio on endpoint"
        description: "Service {{ $labels.application }} - Endpoint {{ $labels.uri }} has >10% client errors"

    # =========================
    # 🚨 PER-URI LATENCY
    # =========================
    - alert: HighLatencyPerURI
      expr: |
        histogram_quantile(0.95,
          sum by(application, uri, le)(
            rate(http_server_requests_seconds_bucket{uri!="/actuator/prometheus"}[5m])
          )
        ) > 1.5
      for: 3m
      labels:
        severity: warning
        service: "{{ $labels.application }}"
      annotations:
        summary: "High latency on endpoint"
        description: "Service {{ $labels.application }} - Endpoint {{ $labels.uri }} is slow"

```
# Apply the following `kubectl apply -f PrometheusRule.yaml`:

👉 Full rule file:


---

# 🔥 Key Alerts Included

### ✅ Errors

* High 5xx error rate
* High 4xx error rate
* Per-URI error spikes (absolute + ratio)

### ✅ Latency

* P95 latency
* P99 latency
* Per-URI latency

### ✅ Traffic

* Traffic spike
* No traffic (service down)

### ✅ Success Rate

* Low success rate

---

# ⚠️ Important Fixes (Must Know)

##  Service Label Issue (VERY COMMON)

❌ Problem:
Slack shows empty `Service`

✅ Fix:

```yaml
group_by: [alertname, application, service, namespace]
```

---


##  Always Use `application` Label

Your metrics contain:

```
application="your-service-name"
```

👉 Use it everywhere:

* Alerts
* Grouping
* Slack messages

---

# 🚀 Final Result

Slack alert example:

```
🚨 HighLatencyPerURI
Severity: warning

Service: esb-shopify-order-integration-stg-xxxxxx
Application: esb-shopify-order-integration-stg-xxxxx
Namespace: monitoring

Description: Endpoint /orders/{id} is slow
```

---

# 🧠 Pro Tips

* Prefer **error ratios over absolute counts**
* Always include labels in `group by`
* Avoid high-cardinality labels (uri, userId, etc.)
* Use `application` as primary service identifier
