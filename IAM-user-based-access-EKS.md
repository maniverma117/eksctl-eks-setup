# 🎯 EKS Role-Based Access Setup (Admin, L2, L1)

* IAM user, group, role creation
* Permissions
* Trust policies
* AWS CLI setup
* Role assumption
* EKS access


## 🛠️ Prerequisites

- EKS cluster already created (`staging-eshopbox`)
- AWS CLI and `kubectl` installed
- Admin credentials to create IAM users, roles, and policies

---

## 1️⃣ IAM Setup

### 1.1 Create IAM Groups

```bash
aws iam create-group --group-name EKS-Admin
````

### 1.2 Create IAM Users

```bash
aws iam create-user --user-name tes-eks-user
aws iam add-user-to-group --user-name tes-eks-user --group-name EKS-Admin
```

### 1.3 Create IAM Role (e.g., `EKSAdminRole`)

#### Trust Policy (`trust-policy.json`)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::98xxxxxxx245:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name EKSAdminRole \
  --assume-role-policy-document file://trust-policy.json
```

### 1.4 Attach Permissions to the Role

#### Policy: `EKSAdminRolePolicy.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EKSBasicAccess",
      "Effect": "Allow",
      "Action": [
        "eks:ListClusters",
        "eks:DescribeCluster"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam put-role-policy \
  --role-name EKSAdminRole \
  --policy-name EKSAdminRoleInlinePolicy \
  --policy-document file://EKSAdminRolePolicy.json
```

### 1.5 Attach AssumeRole Permission to Group

#### Inline policy for group `EKS-Admin`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::989xxxxxxxxxx245:role/EKSAdminRole"
    }
  ]
}
```

```bash
aws iam put-group-policy \
  --group-name EKS-Admin \
  --policy-name AssumeEKSAdminRole \
  --policy-document file://assume-role-policy.json
```

---

## 2️⃣ EKS Cluster RBAC Mapping

### 2.1 Edit `aws-auth` ConfigMap

```bash
kubectl edit -n kube-system configmap/aws-auth
```

Add under `mapRoles`:

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::98xxxxxxxxx5:role/EKSAdminRole
    username: admin
    groups:
      - system:masters
```

---

## 3️⃣ End-User Setup (`tes-eks-user`)

### 3.1 Configure AWS CLI Credentials

Add user credentials to `~/.aws/credentials`:

```ini
[test-user]
aws_access_key_id = <ACCESS_KEY>
aws_secret_access_key = <SECRET_KEY>
```

### 3.2 Configure Profile to Assume Role

Add to `~/.aws/config`:

```ini
[profile test-asumerole-eks]
role_arn = arn:aws:iam::98xxxxxxxxx5:role/EKSAdminRole
source_profile = test-user
region = ap-south-1
```

---

## 4️⃣ Access EKS Cluster

Run the following to set up `kubeconfig`:

```bash
aws eks --region ap-south-1 \
  update-kubeconfig \
  --name staging-eshopbox \
  --profile test-asumerole-eks
```

Verify access:

```bash
kubectl get nodes
```

You should be authenticated as the assumed role and granted full admin access via Kubernetes RBAC.

---

## ✅ Notes

* Do **not** map the IAM user directly in `aws-auth`; only map the IAM **role**.
* Use short-lived credentials by assuming roles instead of long-lived user permissions.
* You can repeat this for `L2` and `L1` roles by:

  * Creating new roles with specific Kubernetes ClusterRoles
  * Updating `aws-auth` and RBAC accordingly
  * Adding matching group-based assume-role permissions

---

## 📁 Files

* `trust-policy.json`
* `EKSAdminRolePolicy.json`
* `assume-role-policy.json`

---

* **IAM setup for L2 and L1 roles**
* **Kubernetes ClusterRoles and RoleBindings**
* **`aws-auth` ConfigMap update**
* **RBAC scoping (no delete for L2, read-only for L1)**

## 5️⃣ Create IAM Roles for L2 and L1

### 5.1 Create IAM Roles

Repeat the same steps as Admin for the following:

#### ➤ L2 Role: `EKSClusterL2Role`

**Trust Policy (`trust-policy-l2.json`):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::98xxxxxxxx245:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
````

**IAM Permissions Policy (`eks-basic-access-l2.json`):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:ListClusters",
        "eks:DescribeCluster"
      ],
      "Resource": "*"
    }
  ]
}
```

#### ➤ L1 Role: `EKSClusterL1Role`

Use the same trust and permission policy as L2.

Then attach to appropriate IAM groups:

```bash
aws iam create-group --group-name EKS-L2
aws iam create-group --group-name EKS-L1

aws iam put-group-policy \
  --group-name EKS-L2 \
  --policy-name AssumeEKSClusterL2Role \
  --policy-document file://assume-role-policy-l2.json

aws iam put-group-policy \
  --group-name EKS-L1 \
  --policy-name AssumeEKSClusterL1Role \
  --policy-document file://assume-role-policy-l1.json
```

---

## 6️⃣ Configure Kubernetes RBAC for L2 and L1

### 6.1 Create ClusterRole for L2 (No Delete Access)

```yaml
# clusterrole-l2.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: eks-l2-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
```

### 6.2 Create ClusterRoleBinding for L2

```yaml
# clusterrolebinding-l2.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eks-l2-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: eks-l2-clusterrole
subjects:
- kind: User
  name: l2-user
  apiGroup: rbac.authorization.k8s.io
```

### 6.3 Create ClusterRole for L1 (Read-Only)

```yaml
# clusterrole-l1.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: eks-l1-clusterrole
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### 6.4 Create ClusterRoleBinding for L1

```yaml
# clusterrolebinding-l1.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eks-l1-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: eks-l1-clusterrole
subjects:
- kind: User
  name: l1-user
  apiGroup: rbac.authorization.k8s.io
```

> Apply these:

```bash
kubectl apply -f clusterrole-l2.yaml
kubectl apply -f clusterrolebinding-l2.yaml
kubectl apply -f clusterrole-l1.yaml
kubectl apply -f clusterrolebinding-l1.yaml
```

---

## 7️⃣ Update `aws-auth` ConfigMap

```bash
kubectl edit -n kube-system configmap/aws-auth
```

Add entries under `mapRoles`:

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::98xxxxxxxx245:role/EKSAdminRole
    username: admin
    groups:
      - system:masters

  - rolearn: arn:aws:iam::98xxxxxxxx245:role/EKSClusterL2Role
    username: l2-user
    groups:
      - eks-l2-group

  - rolearn: arn:aws:iam::98xxxxxxxx245:role/EKSClusterL1Role
    username: l1-user
    groups:
      - eks-l1-group
```

> These `groups` map to the Kubernetes groups referenced in the RBAC bindings (`eks-l2-group`, `eks-l1-group`).

---

## ✅ Summary of Access Levels

| Role             | IAM Group | Access Level        | Kubernetes RBAC Group | ClusterRole        |
| ---------------- | --------- | ------------------- | --------------------- | ------------------ |
| EKSAdminRole     | EKS-Admin | Full admin          | system\:masters       | cluster-admin      |
| EKSClusterL2Role | EKS-L2    | Read/write (no del) | eks-l2-group          | eks-l2-clusterrole |
| EKSClusterL1Role | EKS-L1    | Read-only           | eks-l1-group          | eks-l1-clusterrole |

---

## 📁 Additional Files

* `clusterrole-l2.yaml`
* `clusterrolebinding-l2.yaml`
* `clusterrole-l1.yaml`
* `clusterrolebinding-l1.yaml`
* `trust-policy-l2.json`
* `trust-policy-l1.json`
* `assume-role-policy-l2.json`
* `assume-role-policy-l1.json`

---
