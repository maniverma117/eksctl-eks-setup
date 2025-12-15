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





# 1️⃣ Adding AppProject & Application (Correct + Safe)

## 1.1 AppProject (`cmn`)


```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: cmn
  namespace: argocd
spec:
  description: Project for common studio applications

  # Allowed Git repositories
  sourceRepos:
    - '*'

  # Allowed destination clusters & namespaces
  destinations:
    - namespace: cmn
      server: '*'

  # Namespace-scoped resources
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'

  # Cluster-scoped resources (use carefully)
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
```

### Why this matters

* Prevents accidental cluster-wide access
* Keeps GitOps secure
* Avoids Argo CD validation errors

---

## 1.2 Application (`studio-cmn-los`)

Your Application YAML is **correct and production-ready**.

### Final Polished Version

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: studio-cmn-los
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  labels:
    name: studio-cmn-los
spec:
  project: cmn
  source:
    repoURL: git@bitbucket.org:kulizadev/baobab-helm-chart.git
    targetRevision: SIT
    path: finvolv
    helm:
      valueFiles:
      - studio-cmn-los.yaml
          
  destination:
    # cluster API URL
    server: https://kubernetes.default.svc
    # or cluster name
    # name: in-cluster
    # The namespace will only be set for namespace-scoped resources that have not set a value for .metadata.namespace
    namespace: cmn

  # Sync policy
#  syncPolicy:
#    automated: # automated sync by default retries failed attempts 5 times with following delays between attempts ( 5s, 10s, 20s, 40s, 80s ); retry controlled using `retry` field.
#      prune: true # Specifies if resources should be pruned during auto-syncing ( false by default ).
#      selfHeal: true # Specifies if partial app sync should be executed when resources are changed only in target Kubernetes cluster and no git change detected ( false by default ).
#      allowEmpty: false # Allows deleting all application resources during automatic syncing ( false by default ).
#    syncOptions:     # Sync options which modifies sync behavior
#    - Validate=false # disables resource validation (equivalent to 'kubectl apply --validate=false') ( true by default ).
#    - CreateNamespace=true # Namespace Auto-Creation ensures that namespace specified as the application destination exists in the destination cluster.
#    - PrunePropagationPolicy=foreground # Supported policies are background, foreground and orphan.
#    - PruneLast=true # Allow the ability for resource pruning to happen as a final, implicit wave of a sync operation
#    retry:
#      limit: 5 # number of failed sync attempt retries; unlimited number of attempts if less than 0
#      backoff:
##        duration: 5s # the amount to back off. Default unit is seconds, but could also be a duration (e.g. "2m", "1h")
##        factor: 2 # a factor to multiply the base duration after each failed retry

```

> 💡 You can enable auto-sync later once validation is done.

---

## 1.3 Apply Project & App

```bash
kubectl apply -f appproject-cmn.yaml
kubectl apply -f app-studio-cmn-los.yaml
```

Verify:

```bash
argocd app list
argocd app get studio-cmn-los
```

---

# 2️⃣ Integrating Argo CD with GitHub / Bitbucket

This is the **MOST IMPORTANT part** for seamless deployment.

---

## 🔐 Authentication Options (Choose One)

| Method               | Recommended     | Notes              |
| -------------------- | --------------- | ------------------ |
| SSH key              | ✅ YES           | Best for Bitbucket |
| HTTPS + App Password | ⚠️ OK           | Less secure        |
| GitHub App           | ⭐ BEST (GitHub) | Enterprise-grade   |

---

## 2.1 Bitbucket Integration (Workspace Level – Recommended)

### Step 1: Create SSH Key for Argo CD

```bash
ssh-keygen -t ed25519 -f argocd-bitbucket -C "argocd"
```

Files:

* `argocd-bitbucket` (private)
* `argocd-bitbucket.pub` (public)

---

### Step 2: Add Public Key to Bitbucket

Bitbucket → **Workspace settings** → **SSH keys** → Add key

Paste:

```bash
cat argocd-bitbucket.pub
```

✔ This grants access to **all repos in the workspace**

---

### Step 3: Add Repo to Argo CD

```bash
argocd repo add git@bitbucket.org:kulizadev/baobab-helm-chart.git \
  --ssh-private-key-path argocd-bitbucket
```

Verify:

```bash
argocd repo list
```

---

## 2.2 GitHub Integration (Two Ways)

---

### 🟢 Option A: GitHub App (Enterprise Recommended)

* Create GitHub App
* Grant repo read permissions
* Generate private key
* Configure in Argo CD

👉 Best for large orgs

---

### 🟢 Option B: SSH Key (Simple & Common)

```bash
ssh-keygen -t ed25519 -f argocd-github -C "argocd"
```

Add public key to:

* Repo → Settings → Deploy Keys → **Read only**

Add to Argo CD:

```bash
argocd repo add git@github.com:org/repo.git \
  --ssh-private-key-path argocd-github
```

---

# 3️⃣ End-to-End Deployment Flow (How It Actually Works)

## 🔄 Complete GitOps Flow

```
Developer
  |
  | git push
  ▼
GitHub / Bitbucket
  |
  | webhook / polling
  ▼
Argo CD
  |
  | helm template + diff
  ▼
Target Kubernetes Cluster
```

---

## Step-by-Step Lifecycle

### 1️⃣ Developer commits Helm values

```bash
git commit -am "Update studio-cmn-los config"
git push origin SIT
```

---

### 2️⃣ Argo CD detects change

* Polls Git every 3 mins (default)
* Or webhook (optional)

---

### 3️⃣ Argo CD renders Helm

```bash
helm template finvolv -f studio-cmn-los.yaml
```

---

### 4️⃣ Argo CD compares desired vs live

* Drift detected
* Status → `OutOfSync`

---

### 5️⃣ Sync (Manual or Auto)

```bash
argocd app sync studio-cmn-los
```

or auto-sync (if enabled)

---

### 6️⃣ Kubernetes resources updated

* Deployments
* Services
* ConfigMaps
* Secrets (if Git-managed)

---

## 4️⃣ Recommended Enhancements (Next Level)

### ✔ Enable Webhooks

* Faster deployments
* Less Git polling

### ✔ Use App-of-Apps

```text
gitops/
├── projects/
├── apps/
│   └── cmn/
└── clusters/
```

### ✔ Restrict Projects

* One project per team
* One namespace per project

---

## 5️⃣ Final Summary (Crystal Clear)

✔ AppProject controls **who can deploy where**
✔ Application links **Git → Cluster → Namespace**
✔ SSH keys give **secure Git access**
✔ Bitbucket workspace key scales best
✔ Git push = deployment


