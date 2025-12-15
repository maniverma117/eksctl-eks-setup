# Argo CD – Centralized GitOps Setup (Multi-Cluster)

This document describes how Argo CD is installed using Helm and how multiple Kubernetes clusters (k3s, EKS, GKE, AKS) are added and managed centrally.

---

## 1. Overview

* Argo CD is installed on a **central Kubernetes cluster**
* Helm is used for **upgrade-safe installation**
* Argo CD manages **remote Kubernetes clusters**
* Git is the **single source of truth**

---

## 2. Prerequisites

### Tools Required

* Helm v3+
* kubectl
* argocd CLI
* Network connectivity from Argo CD to target cluster APIs

### Install Argo CD CLI

```bash
curl -sSL -o /usr/local/bin/argocd \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

---

## 3. Helm Repository Setup

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Verify available charts:

```bash
helm search repo argo
```

---

## 4. Install Argo CD Using Helm

### Create Namespace

```bash
kubectl create namespace argocd
```

### Install with Values File

```bash
helm install argocd argo/argo-cd \
  -n argocd \
  -f node-select-values.yaml
```

### Upgrade (future-safe)

```bash
helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  -f node-select-values.yaml
```

---

## 5. Login to Argo CD

Login using the internal service DNS (from cluster/bastion):

```bash
argocd login argocd-server.argocd.svc.cluster.local --insecure
```

Get admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
-o jsonpath="{.data.password}" | base64 -d && echo
```

---

## 6. Adding Clusters to Argo CD

> **Important**
> Cluster access is stored as Kubernetes Secrets in the Argo CD namespace.
> Kubeconfig files are used **only once** during cluster registration.

---

## 6.1 Add k3s Cluster

### Prepare kubeconfig

* Copy `/etc/rancher/k3s/k3s.yaml`
* Replace `127.0.0.1` with reachable node IP

Verify:

```bash
kubectl --kubeconfig k3s.yaml get nodes
```

### Add cluster

```bash
argocd cluster add default \
  --kubeconfig k3s.yaml \
  --insecure
```

Verify:

```bash
argocd cluster list
```

---

## 6.2 Add Amazon EKS Cluster

### Prerequisites

* AWS CLI configured
* IAM access to EKS

### Fetch kubeconfig

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name <EKS_CLUSTER_NAME>
```

Verify:

```bash
kubectl get nodes
```

### Add to Argo CD

```bash
argocd cluster add <EKS_CONTEXT_NAME>
```

Example:

```bash
argocd cluster add arn:aws:eks:ap-south-1:123456789012:cluster/prod-eks
```

---

## 6.3 Add Google GKE Cluster

### Prerequisites

* gcloud CLI authenticated
* GKE permissions

### Fetch kubeconfig

```bash
gcloud container clusters get-credentials <CLUSTER_NAME> \
  --region <REGION> \
  --project <PROJECT_ID>
```

Verify:

```bash
kubectl get nodes
```

### Add to Argo CD

```bash
argocd cluster add <GKE_CONTEXT_NAME>
```

---

## 6.4 Add Azure AKS Cluster

### Prerequisites

* Azure CLI authenticated
* AKS access

### Fetch kubeconfig

```bash
az aks get-credentials \
  --resource-group <RESOURCE_GROUP> \
  --name <AKS_CLUSTER_NAME>
```

Verify:

```bash
kubectl get nodes
```

### Add to Argo CD

```bash
argocd cluster add <AKS_CONTEXT_NAME>
```

---

## 7. Verify Cluster Registration

```bash
argocd cluster list
```

Expected:

```
SERVER                                 NAME        STATUS
https://x.x.x.x:6443                  k3s-prod    Successful
https://ABC.eks.amazonaws.com         eks-prod    Successful
https://gke.googleapis.com/...        gke-prod    Successful
https://aks-api-server                aks-prod    Successful
```

---

## 8. Important Operational Notes

### Cluster Persistence

* Cluster credentials are stored as Secrets
* Restarting Argo CD pods does **not** require re-adding clusters
* Only deleting the `argocd` namespace removes cluster access

### One Argo CD per Cluster Rule

> Only **one Argo CD instance** should manage a given cluster at a time

---

## 9. Best Practices

* Rename kubeconfig contexts (avoid `default`)
* Disable auto-sync during migrations
* Store Applications & Projects in Git
* Use App-of-Apps pattern
* Restrict cluster RBAC where possible

---

## 10. Summary

* Helm-based Argo CD installation is upgrade-safe
* Clusters (k3s, EKS, GKE, AKS) can be centrally managed
* Kubeconfig files are not required after initial registration
* Git remains the single source of truth
