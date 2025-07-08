# Loki Stack Deployment on AWS EKS

This guide provides step-by-step instructions to deploy **Loki** and **Promtail** on an **AWS EKS** cluster using **Helm**. This setup includes persistence and log retention, with node affinity for scheduling pods on specific nodes.

---

## Prerequisites

* AWS EKS cluster up and running
* `kubectl` configured to communicate with your EKS cluster
* Helm installed (`helm version`)
* A storage class named `ebs-sc` created for persistent volumes

---

## Step 1: Add Grafana Helm Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## Step 2: Download Loki Stack Chart

```bash
helm pull grafana/loki-stack --untar
```

---

## Step 3: Create Custom `loki-values.yaml`

This file customizes Loki configuration for:

* Retention period: 28 days
* Persistent volume: 150Gi using `ebs-sc` StorageClass
* Node affinity to schedule on nodes with `role=Prd-Monitoring-NG-1`
* Promtail enabled to collect logs

Create the file:

```bash
cat << EOF > loki-values.yaml
loki:
  config:
    table_manager:
      retention_deletes_enabled: true
      retention_period: 672h  # 28 days = 672 hours
  persistence:
    enabled: true
    storageClassName: ebs-sc
    accessModes:
      - ReadWriteOnce
    size: 150Gi

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
EOF
```

---

## Step 4: Deploy Loki Stack

First, perform a dry run to validate the configuration:

```bash
helm upgrade --install loki -f loki-values.yaml loki-stack/ -n monitoring --create-namespace --dry-run
```

If the dry run is successful, proceed to deploy:

```bash
helm upgrade --install loki -f loki-values.yaml loki-stack/ -n monitoring --create-namespace
```

---

## Post-Deployment

* Check the pods in the `monitoring` namespace:

  ```bash
  kubectl get pods -n monitoring
  ```

* Verify persistent volume claims:

  ```bash
  kubectl get pvc -n monitoring
  ```

* Integrate with Grafana by adding Loki as a data source.

---

## Notes

* Ensure the nodes labeled with `role=Prd-Monitoring-NG-1` are part of your EKS cluster.
* You may customize the retention period and volume size as per your requirements.
* The `ebs-sc` StorageClass must exist in your cluster.


