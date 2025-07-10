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
