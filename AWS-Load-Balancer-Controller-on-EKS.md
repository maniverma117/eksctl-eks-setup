# 🚀 Deploying AWS Load Balancer Controller on EKS

This guide outlines the steps to install the AWS Load Balancer Controller on the EKS cluster in the `ap-south-1` region using **IAM OIDC**, **custom IAM policy**, and **Helm**.

---

## 🔧 Prerequisites

* EKS cluster: `Non-Prod-EduNext` is already created.
* IAM OIDC is not yet associated.
* AWS CLI, `eksctl`, `kubectl`, and `helm` are installed.
* IAM permissions to create roles and policies in account `92xcxxxxx943`.

---

## ✅ Steps

---

### 1️⃣ Associate IAM OIDC Provider

This step allows the EKS cluster to use IAM roles for Kubernetes service accounts (IRSA).

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster Non-Prod-EduNext \
  --approve
```

---

### 2️⃣ Create IAM Policy for Load Balancer Controller

Download and create the IAM policy required by the AWS Load Balancer Controller:

```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create the policy:

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

⚠️ **Note the policy ARN returned**, e.g.:

```
arn:aws:iam::92xcxxxxx943:policy/AWSLoadBalancerControllerIAMPolicy
```

---

### 3️⃣ Create IAM Service Account in EKS

Create the Kubernetes service account with attached IAM policy:

```bash
eksctl create iamserviceaccount \
  --cluster=Non-Prod-EduNext \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::92xcxxxxx943:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region ap-south-1 \
  --approve
```

This creates a service account with IAM permissions to manage AWS Load Balancers.

---

### 4️⃣ Install AWS Load Balancer Controller using Helm

Add the AWS EKS Helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install the controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=Non-Prod-EduNext \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## 📌 Verification

After deployment, verify the controller is running:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

It should show `READY 1/1` and `AVAILABLE`.

---

## 📚 Notes

* The ALB controller uses `Ingress` resources to provision and manage AWS ALBs.
* The controller must be deployed in the same namespace as the service account (`kube-system`).
* You can now create Kubernetes `Ingress` objects, and the controller will provision ALBs for them.

---

## ✅ Summary

| Step | Description                       |
| ---- | --------------------------------- |
| ✅    | OIDC Provider Associated          |
| ✅    | IAM Policy Created                |
| ✅    | IAM Service Account Created       |
| ✅    | ALB Controller Installed via Helm |
