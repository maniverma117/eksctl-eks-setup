# Kube-Prometheus-Stack Deployment on AWS EKS


This repository contains the setup instructions and Helm configuration to deploy the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) on an AWS EKS cluster. The stack is configured to run on a dedicated monitoring node with persistent storage for Prometheus, Grafana, and Alertmanager.



🚀 Prerequisites

- AWS EKS Cluster
- `kubectl` configured to access the cluster
- Helm 3 installed
- A node labeled with: `role=Prd-Monitoring-NG-1`
- A StorageClass named `ebs-sc` (Amazon EBS)

---

🛠️ Setup Instructions

### 1. Add Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
````

### 2. Pull and Untar the Chart

```bash
helm pull prometheus-community/kube-prometheus-stack --untar
```

### 3. Create Custom Values File

Create a file named `monitoring-values.yaml`:

```bash
cat << EOF > monitoring-values.yaml
prometheus:
  prometheusSpec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: role
                  operator: In
                  values:
                    - Prd-Monitoring-NG-1
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 100Gi
          storageClassName: ebs-sc

alertmanager:
  alertmanagerSpec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: role
                  operator: In
                  values:
                    - Prd-Monitoring-NG-1
    storage:
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 20Gi
          storageClassName: ebs-sc

grafana:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values:
                  - Prd-Monitoring-NG-1
  persistence:
    enabled: true
    type: pvc
    storageClassName: ebs-sc
    accessModes:
      - ReadWriteOnce
    size: 50Gi
EOF
```

---

### 4. Dry Run to Validate the Deployment

```bash
helm upgrade --install edunext \
  -f monitoring-values.yaml \
  -f kube-prometheus-stack/values.yaml \
  kube-prometheus-stack/ \
  -n monitoring --create-namespace \
  --dry-run
```

---

### 5. Deploy to the Cluster

```bash
helm upgrade --install edunext \
  -f monitoring-values.yaml \
  -f kube-prometheus-stack/values.yaml \
  kube-prometheus-stack/ \
  -n monitoring --create-namespace
```

---

## ✅ Post-Deployment

Check the pods and confirm they are scheduled on the correct node:

```bash
kubectl get pods -n monitoring -o wide
```

Verify PVCs are created and bound:

```bash
kubectl get pvc -n monitoring
```

---

## 📎 Notes

* The node label `role=Prd-Monitoring-NG-1` must be applied before deployment:

  ```bash
  kubectl label node <your-node-name> role=Prd-Monitoring-NG-1
  ```
* The `ebs-sc` storage class should exist. You can check available storage classes with:

  ```bash
  kubectl get storageclass
  ```
