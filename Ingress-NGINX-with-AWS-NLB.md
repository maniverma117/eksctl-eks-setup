# Ingress NGINX with AWS NLB (TLS Termination + Proxy Protocol v1)

**Helm Chart:** `ingress-nginx-4.14.2`
**Controller Version:** `ingress-nginx/controller:v1.14.2`
**Load Balancer:** AWS Network Load Balancer (NLB)
**TLS Termination:** At NLB (ACM)
**Redirect:** HTTP → HTTPS at controller level
**Client IP Preservation:** Enabled (Proxy Protocol v1)

---
## Helm Chart ingress-nginx

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm pull ingress-nginx/ingress-nginx --untar
```
## vim my-value.yaml
```
global:
  image:
    registry: registry.k8s.io

controller:
  name: controller

  image:
    image: ingress-nginx/controller
    tag: "v1.14.2"
    pullPolicy: IfNotPresent
    runAsNonRoot: true
    runAsUser: 101
    runAsGroup: 82
    allowPrivilegeEscalation: false

  kind: DaemonSet
  replicaCount: 1

  ingressClass: nginx
  ingressClassResource:
    name: nginx
    enabled: true
    default: false
    controllerValue: k8s.io/ingress-nginx

  allowSnippetAnnotations: true

  config:
    use-proxy-protocol: "true"
    ssl-redirect: "true"
    force-ssl-redirect: "true"
    proxy-protocol-version: "1"
    use-forwarded-headers: "true"
    real-ip-header: "proxy_protocol"
    proxy-real-ip-cidr: "10.69.0.0/16"
    http-redirect-code: "308"

  containerPort:
    http: 80
      # https: 443

  publishService:
    enabled: true

  service:
    enabled: true
    type: LoadBalancer
    externalTrafficPolicy: Local

    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"   
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "300"
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:ap-south-1:989064034245:certificate/c6ee4c85-bad3-4399-b06d-9b3b3f1fea31"
      service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
      service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-0ae16a7a1fe949602,subnet-0fc0a16eecc882171"
      service.beta.kubernetes.io/aws-load-balancer-name: "eshopbox-nlb-prod"

    enableHttp: true
    enableHttps: true

    ports:
      http: 80
      https: 443

    targetPorts:
      http: http
      https: http

  nodeSelector:
    kubernetes.io/os: linux

  resources:
    requests:
      cpu: 100m
      memory: 90Mi

  livenessProbe:
    httpGet:
      path: /healthz
      port: 10254
    initialDelaySeconds: 10
    periodSeconds: 10

  readinessProbe:
    httpGet:
      path: /healthz
      port: 10254
    initialDelaySeconds: 10
    periodSeconds: 10

  metrics:
    enabled: true
    service:
      enabled: true
      type: ClusterIP
      servicePort: 10254

  lifecycle:
    preStop:
      exec:
        command:
          - /wait-shutdown

defaultBackend:
  enabled: false

rbac:
  create: true

serviceAccount:
  create: true
  automountServiceAccountToken: true

tcp: {}
udp: {}

imagePullSecrets: []

revisionHistoryLimit: 10

```


## 📌 Overview

This setup describes a **production-grade ingress-nginx deployment** on Kubernetes using:

* **AWS Network Load Balancer (NLB)**
* **TLS termination at NLB**
* **Proxy Protocol v1** to preserve original request metadata
* **Controller-level HTTP → HTTPS redirection**
* **DaemonSet-based ingress controller** for node-local traffic handling

This configuration is especially useful when:

* You **must use NLB** (not ALB)
* You want **HTTPS redirection at ingress/controller level**
* You want **real client IPs**
* You are upgrading from **older ingress-nginx versions** where redirects used to “just work”

---

## 🧠 High-Level Traffic Flow

```
Client (HTTP/HTTPS)
   ↓
AWS NLB (TLS termination, Proxy Protocol v1)
   ↓
ingress-nginx (HTTP only, port 80)
   ↓
Kubernetes Services
   ↓
Application Pods
```

Key idea:

* **Ingress never handles TLS**
* **Ingress always listens on HTTP**
* **Ingress still knows whether the original request was HTTPS** (via Proxy Protocol)

---

## 🔑 Why Proxy Protocol Is Critical Here

When TLS is terminated at the **NLB**, ingress-nginx would normally see *all traffic as HTTP*.
Without extra metadata, it **cannot distinguish**:

* real HTTP requests
* HTTPS requests that were terminated upstream

This breaks:

* `ssl-redirect`
* `force-ssl-redirect`

### ✅ Solution

Enable **Proxy Protocol v1** on:

* **AWS NLB**
* **ingress-nginx controller**

This allows ingress-nginx to reconstruct:

* Original client IP
* Original destination port (80 vs 443)
* Original scheme (HTTP vs HTTPS)

---

## 🚨 Important Version Change (Why This Broke After Upgrade)

| Component     | Old Behavior                | New Behavior                      |
| ------------- | --------------------------- | --------------------------------- |
| ingress-nginx | Implicit Proxy Protocol v1  | Defaults to **Proxy Protocol v2** |
| AWS NLB       | Sends **Proxy Protocol v1** | Still v1                          |
| Result        | Worked                      | ❌ `ERR_EMPTY_RESPONSE`            |

### 🔥 Fix

Explicitly set:

```yaml
proxy-protocol-version: "1"
```

This single line restores compatibility.

---

## 📦 Helm Values Explained (Section by Section)

---

### 🌍 Global Image Settings

```yaml
global:
  image:
    registry: registry.k8s.io
```

* Uses the official Kubernetes image registry
* Ensures trusted and signed images

---

### 🚀 Controller Image & Security Context

```yaml
controller:
  image:
    image: ingress-nginx/controller
    tag: "v1.14.2"
```

Security hardening:

```yaml
runAsNonRoot: true
runAsUser: 101
runAsGroup: 82
allowPrivilegeEscalation: false
```

✅ Best practice for production clusters
✅ Required for restricted PSP / PSA environments

---

### 🧩 Deployment Model

```yaml
kind: DaemonSet
replicaCount: 1
```

Why **DaemonSet**?

* One ingress pod per node
* Works perfectly with:

  * `externalTrafficPolicy: Local`
  * NLB target type = `instance`
* Preserves **client source IP**

---

### 🚦 IngressClass Configuration

```yaml
ingressClass: nginx
ingressClassResource:
  name: nginx
  enabled: true
  default: false
  controllerValue: k8s.io/ingress-nginx
```

* Explicit ingress class
* Avoids conflicts with other controllers
* Safe for multi-ingress environments

---

### 🧠 Core Controller Configuration (MOST IMPORTANT)

```yaml
config:
  use-proxy-protocol: "true"
  proxy-protocol-version: "1"
```

🔑 **This is the heart of the solution**

* Tells ingress-nginx to:

  * Expect Proxy Protocol
  * Parse **version 1**, matching AWS NLB

---

#### 🔐 HTTPS Redirection (Controller-Level)

```yaml
ssl-redirect: "true"
force-ssl-redirect: "true"
http-redirect-code: "308"
```

* Forces HTTP → HTTPS
* Uses **308 Permanent Redirect** (safe for POST, PUT, etc.)
* Works **only because Proxy Protocol is enabled**

---

#### 🌐 Client IP Handling

```yaml
use-forwarded-headers: "true"
real-ip-header: "proxy_protocol"
proxy-real-ip-cidr: "10.69.0.0/16"
```

* Extracts real client IP from Proxy Protocol
* CIDR must match **node / VPC CIDR**
* Required for:

  * Logging
  * Rate limiting
  * Security rules

---

### 🔌 Container Ports

```yaml
containerPort:
  http: 80
```

* Ingress listens **only on HTTP**
* No internal TLS
* TLS is handled **exclusively by NLB**

---

### 🌐 Service Configuration (AWS NLB)

```yaml
service:
  type: LoadBalancer
  externalTrafficPolicy: Local
```

Why `Local`?

* Preserves client IP
* Required for Proxy Protocol + DaemonSet

---

#### 🏗️ NLB Annotations (CRITICAL)

```yaml
service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
```

Key points:

* `proxy-protocol: "*"` → enables Proxy Protocol v1
* `backend-protocol: tcp` → no HTTP termination at LB
* Target type `instance` → required for DaemonSet

---

#### 🔐 TLS at NLB

```yaml
service.beta.kubernetes.io/aws-load-balancer-ssl-cert: <ACM_ARN>
service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
```

* TLS terminated at NLB
* ACM-managed certificate
* Ingress never sees encrypted traffic

---

### 🔁 Port Mapping (Subtle but Important)

```yaml
ports:
  http: 80
  https: 443

targetPorts:
  http: http
  https: http
```

Meaning:

* Both 80 and 443 on the NLB
* Forward to **port 80** in ingress
* Ingress decides redirect based on **Proxy Protocol metadata**

---

### ❤️ Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 10254
```

* Uses ingress-nginx internal health endpoint
* Safe with NLB health checks

---

### 📊 Metrics

```yaml
metrics:
  enabled: true
  service:
    type: ClusterIP
    servicePort: 10254
```

* Enables Prometheus scraping
* Internal-only metrics endpoint

---

## 🧪 Final Expected Behavior

```bash
curl -I http://scheduler.eshpbx.com
```

➡️ `308 Permanent Redirect`

```bash
curl -I https://scheduler.eshpbx.com
```

➡️ `200 OK`

Browser:

* 🔒 Secure
* No redirect loops
* Real client IP preserved
* Works exactly like older clusters

---

## 🧾 Key Takeaways

* **NLB + TLS termination breaks HTTPS detection by default**
* **Proxy Protocol restores original request context**
* **New ingress-nginx versions require explicit proxy protocol version**
* `proxy-protocol-version: "1"` is the missing link
* This setup is **stable, secure, and production-proven**

---

## ✅ When to Use This Pattern

Use this **exact configuration** when:

* You must use **AWS NLB**
* You need **controller-level HTTPS redirect**
* You want **real client IPs**
* You’re upgrading ingress-nginx and redirects suddenly break

