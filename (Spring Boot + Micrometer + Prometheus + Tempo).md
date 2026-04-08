# 📊 Observability Setup Guide (Spring Boot + Micrometer + Prometheus + Tempo)

This guide explains how to enable **URI-level metrics**, **latency histograms**, and **trace correlation** for all services.

---

# 🚀 Goal

Every service should provide:

* ✅ Request count (per API)
* ✅ Latency (avg, p95, p99)
* ✅ Error rate
* ✅ URI-level breakdown (`/users`, `/orders`)
* ✅ Correlation with traces (Tempo)

---

# 🧱 Tech Stack

* Spring Boot 3.x
* Micrometer
* Prometheus
* Grafana
* OpenTelemetry Java Agent
* Tempo
* Loki (logs)

---

# 📦 1. Add Dependencies (pom.xml)

```xml
<dependencies>

    <!-- Spring Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Actuator (required for metrics endpoint) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Prometheus Registry -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

</dependencies>
```

---

# ⚙️ 2. Application Properties

Add the following to `application.properties`:

```properties
spring.application.name=app2
logging.pattern.level=%5p [trace_id=%X{trace_id} span_id=%X{span_id}]

# Enable actuator endpoints
management.endpoints.web.exposure.include=prometheus,health,info

# Enable prometheus endpoint
management.endpoint.prometheus.enabled=true

# Enable metrics
management.metrics.export.prometheus.enabled=true

# IMPORTANT: Enable histogram (for latency, p95, etc.)
management.metrics.distribution.percentiles-histogram.http.server.requests=true

# Optional but recommended (better visibility)
management.metrics.distribution.percentiles.http.server.requests=0.5,0.9,0.95,0.99

# Add common tag (helps in Grafana filtering)
management.metrics.tags.application=app1

# Ensure all HTTP requests are timed
management.metrics.web.server.request.autotime.enabled=true

management.metrics.web.server.request.ignore-trailing-slash=true

```

---

# 🔥 3. Controller Best Practices (CRITICAL)

Metrics depend on proper Spring mappings.

✅ Correct:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping
    public String getUsers() {
        return "users";
    }
}
```

❌ Avoid:

* Raw Servlets
* Filters bypassing Spring MVC
* Unmapped endpoints

---

# 🚫 4. Remove UNKNOWN URIs (Optional but Recommended)

Create file:

```
src/main/java/com/yourpackage/MetricsConfig.java
```

```java
package com.demo.app1;

import io.micrometer.core.instrument.config.MeterFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MetricsConfig {

    @Bean
    public MeterFilter ignoreUnknownUri() {
        return MeterFilter.deny(id -> {
            String uri = id.getTag("uri");
            return uri != null && uri.equals("UNKNOWN");
        });
    }
}
```

---

# 🔍 5. Verify Metrics

Run the app and check:

```
http://<host>:<port>/actuator/prometheus
```

You should see:

```
http_server_requests_seconds_count{uri="/users"}
http_server_requests_seconds_bucket{uri="/users"}
```

❌ If you see `uri="UNKNOWN"` → controller mapping issue

---

# ☸️ 6. Kubernetes Service Requirements

Your Service MUST have:

```yaml
metadata:
  labels:
    app: app1

spec:
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

---

# 📡 7. ServiceMonitor (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
    meta.helm.sh/release-name: esb-shopify-integration-stg
    meta.helm.sh/release-namespace: stg
  labels:
    app.kubernetes.io/managed-by: Helm
    release: prometheus
  name: esb-shopify-integration-stg-eshopbox
  namespace: stg
spec:
  endpoints:
  - interval: 30s
    path: /actuator/prometheus
    port: http
  namespaceSelector:
    matchNames:
    - stg
  selector:
    matchLabels:
      application: esb-shopify-integration-stg
      helm.sh/chart: eshopbox-0.1.0
      managed-by: Velocis
      project: eshopbox
      version: 1.16.0
```

---

# 🔗 8. Trace Correlation (IMPORTANT)

Already enabled via OpenTelemetry Java Agent.

Ensure:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

---

# 🔥 9. Prometheus Queries (Grafana)

### Requests/min

```promql
sum(rate(http_server_requests_seconds_count[1m])) by (uri)
```

### Latency (avg)

```promql
sum(rate(http_server_requests_seconds_sum[5m])) by (uri)
/
sum(rate(http_server_requests_seconds_count[5m])) by (uri)
```

### P95 latency

```promql
histogram_quantile(0.95,
  sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri)
)
```

---

# ⚠️ Common Issues

| Issue                 | Cause                         |
| --------------------- | ----------------------------- |
| No data in Prometheus | ServiceMonitor label mismatch |
| Target not visible    | Wrong port name               |
| URI = UNKNOWN         | Controller issue              |
| No latency            | Histogram not enabled         |

---

# 🎯 Final Outcome

Each service will provide:

* `/users` → requests, latency, errors
* `/orders` → requests, latency, errors

All visible in Grafana with trace linking.

---

# 🚀 Result Architecture

```
App (Micrometer)
   ↓
/actuator/prometheus
   ↓
Prometheus
   ↓
Grafana (Dashboard)
   ↓
Tempo (Traces)
```

---

# ✅ Done

After setup, your service is fully observable and production-ready.

---
