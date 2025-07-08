# 🛠️ EKS Production Cluster Setup with Encrypted Volumes, OIDC, and ALB Controller

This guide documents the end-to-end setup of a **production-ready EKS cluster** using `eksctl`, including:

* Private/public subnet configuration
* Encrypted EBS volumes with custom KMS key
* ALB Controller IAM role
* CloudWatch logging
* Best practices for IAM roles and security

---

## 📦 Prerequisites

### 🔧 Tools

* AWS CLI configured with access to the target account (`9890xxxxx245`)
* `eksctl` v0.170.0+ installed
* `kubectl` installed and configured
* `jq` (optional for CLI parsing)

---

## 1️⃣ Step-by-Step Setup

---

### 1.1 Associate OIDC Provider (Once per EKS cluster)

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster staging-eshopbox \
  --approve
```

---

### 1.2 Create KMS Key for Secrets and Volume Encryption

Create the KMS key manually or use AWS CLI, then attach this **KMS key policy**:

```json
{
  "Version": "2012-10-17",
  "Id": "eks-ebs-safe-policy",
  "Statement": [
    {
      "Sid": "EnableRootAccountAdminAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::9890xxxxx245:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowAccessViaEC2EBS",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:CreateGrant",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "ec2.ap-south-1.amazonaws.com",
          "kms:CallerAccount": "9890xxxxx245"
        }
      }
    }
  ]
}
```

📌 This ensures EC2 (and hence EKS nodes) launched by Auto Scaling have access to the key.

---

### 1.3 Create Custom IAM Policy for NodeGroup KMS Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUseOfEBSKMSKey",
      "Effect": "Allow",
      "Action": [
        "kms:CreateGrant",
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:GenerateDataKeyWithoutPlainText",
        "kms:ReEncrypt*"
      ],
      "Resource": "arn:aws:kms:ap-south-1:9890xxxxx245:key/98128a97-d8dd-412c-a9e2-7ba363e53ca2"
    }
  ]
}
```

📝 Save as `staging-eshopbox-kms` and attach to NodeGroup role in `eksctl` config.

---

### 1.4 Create IAM Policy for ALB Controller (Optional Step)

If you’re using the AWS ALB Ingress Controller, create a policy like:

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy-StagingEshopbox \
  --policy-document file://iam_policy.json
```

---

## 2️⃣ Deploy the EKS Cluster with `eksctl`

> Use the following `eksctl` manifest (`eksctl-staging-cluster.yaml`):

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: staging-eshopbox
  region: ap-south-1
  version: "1.33"

vpc:
  id: "vpc-05319ad9f6bbb90c3"
  cidr: "10.70.0.0/16"
  subnets:
    public:
      ESHOPBOX-STG-Pub-1a:
        id: "subnet-01cf350f528fd179e"
        az: ap-south-1a
      ESHOPBOX-STG-Pub-1b:
        id: "subnet-0487bbc5d1a9134ab"
        az: ap-south-1b
    private:
      ESHOPBOX-STG-Pvt-1a:
        id: "subnet-05f18334383d3c316"
        az: ap-south-1a
      ESHOPBOX-STG-Pvt-1b:
        id: "subnet-073487b0798f7a6ca" #
        az: ap-south-1b

secretsEncryption:
  keyARN: "arn:aws:kms:ap-south-1:9890xxxxx245:key/98128a97-d8dd-412c-a9e2-7ba363e53ca2" # Replace with actual KMS Key ID


iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: aws-load-balancer-controller
        namespace: kube-system
      attachPolicyARNs:
        - arn:aws:iam::9890xxxxx245:policy/AWSLoadBalancerControllerIAMPolicy-StagingEshopbox
      roleName: staging-eshopbox-AWSLoadBalancerController-Role

managedNodeGroups:
  - name: CTRL
    minSize: 1
    maxSize: 2
    desiredCapacity: 1
    instanceType: "m6a.xlarge"
    volumeSize: 30
    volumeEncrypted: true
    volumeKmsKeyID: "98128a97-d8dd-412c-a9e2-7ba363e53ca2"
    privateNetworking: true
    subnets:
      - ESHOPBOX-STG-Pvt-1a
      - ESHOPBOX-STG-Pvt-1b
    ssh:
      publicKeyName: staging-eshopbox-node
    labels: {role: ctrl}
    tags:
      nodegroup-role: ctrl
      nodegroup-name: ctrl
      Project: staging-eshopbox
      Env: staging
      Layer: App
      ManagedBy: Velocis
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        - arn:aws:iam::9890xxxxx245:policy/staging-eshopbox-kms
      withAddonPolicies:
        autoScaler: true
        externalDNS: true
        certManager: true
        ebs: true
        efs: true
        awsLoadBalancerController: true
        cloudWatch: true
```

```bash
eksctl create cluster -f eksctl-staging-cluster.yaml
```

---

## 3️⃣ Post-Create Steps

---

### 3.1 Install AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=staging-eshopbox \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=vpc-05319ad9f6bbb90c3 \
  --set image.tag="v2.7.1"
```

---

### 3.2 Enable CloudWatch Logging

(Optional, if not using a FluentBit or OpenSearch-based solution)

---

## ✅ Summary

| Component             | Configured? |
| --------------------- | ----------- |
| VPC/Subnets           | ✅           |
| Private NodeGroup     | ✅           |
| Encrypted EBS Volumes | ✅           |
| KMS Key Access        | ✅           |
| IAM Roles/OIDC        | ✅           |
| ALB Controller Role   | ✅           |
| CloudWatch Agent      | ✅           |

---

## 🔒 Security Best Practices

* Use `withOIDC: true` + `serviceAccounts` for least-privilege access to AWS services.
* Encrypt secrets using the `secretsEncryption` block.
* Use private subnets for all node groups.
* Use managed policies for base roles, and custom policies only when needed.

---
