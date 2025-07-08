# Installing **Karpenter** on **EKS cluster**

```markdown
# Karpenter Installation Guide for Amazon EKS

This document outlines the steps to install and configure [Karpenter](https://karpenter.sh/) to automatically scale nodes in your Amazon EKS cluster.

## 📋 Prerequisites

- An existing EKS cluster
- `kubectl` and `helm` installed
- AWS CLI configured with sufficient IAM permissions
- ECR public login access
- K8s version ≥ 1.24

---

## 🔧 Step 1: Set Environment Variables

```bash
export KARPENTER_NAMESPACE=kube-system
export CLUSTER_NAME=<your-cluster-name>
export AWS_ACCOUNT_ID=<your-aws-account-id>
export AWS_REGION=<your-region>                     # e.g. ap-south-1
export AWS_PARTITION="aws"                          # default unless in GovCloud
export K8S_VERSION=<your-k8s-version>               # e.g. 1.32
export IMAGE_ID=<ami-id>                            # AMI for worker nodes
export OIDC_ENDPOINT=$(aws eks describe-cluster --name "${CLUSTER_NAME}" --query "cluster.identity.oidc.issuer" --output text)
```

You can retrieve the EKS optimized AMI for ARM:

```bash
aws ssm get-parameter \
  --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/arm64/standard/recommended/image_id" \
  --query "Parameter.Value" \
  --output text
```

Or for AMD64:

```bash
aws ssm get-parameter \
  --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" \
  --query "Parameter.Value" \
  --output text
```

---

## 👥 Step 2: Create Node IAM Role for Karpenter

```bash
cat <<EOF > node-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file://node-trust-policy.json

# Attach required managed policies
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSWorkerNodePolicy"
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKS_CNI_Policy"
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly"
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonSSMManagedInstanceCore"
```

---

## 🛡️ Step 3: Create Controller IAM Role

```bash
cat <<EOF > controller-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
        "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:${KARPENTER_NAMESPACE}:karpenter"
      }
    }
  }]
}
EOF
```
Attach trust policy:

```bash
aws iam create-role \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --assume-role-policy-document file://controller-trust-policy.json
```


```bash
cat << EOF > controller-policy.json
{
    "Statement": [
        {
            "Action": [
                "ssm:GetParameter",
                "ec2:DescribeImages",
                "ec2:RunInstances",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInstanceTypeOfferings",
                "ec2:DeleteLaunchTemplate",
                "ec2:CreateTags",
                "ec2:CreateLaunchTemplate",
                "ec2:CreateFleet",
                "ec2:DescribeSpotPriceHistory",
                "pricing:GetProducts"
            ],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "Karpenter"
        },
        {
            "Action": "ec2:TerminateInstances",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/karpenter.sh/nodepool": "*"
                }
            },
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "ConditionalEC2Termination"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}",
            "Sid": "PassNodeIAMRole"
        },
        {
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "arn:${AWS_PARTITION}:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${CLUSTER_NAME}",
            "Sid": "EKSClusterEndpointLookup"
        },
        {
            "Sid": "AllowScopedInstanceProfileCreationActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
            "iam:CreateInstanceProfile"
            ],
            "Condition": {
            "StringEquals": {
                "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:RequestTag/topology.kubernetes.io/region": "${AWS_REGION}"
            },
            "StringLike": {
                "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
            }
            }
        },
        {
            "Sid": "AllowScopedInstanceProfileTagActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
            "iam:TagInstanceProfile"
            ],
            "Condition": {
            "StringEquals": {
                "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:ResourceTag/topology.kubernetes.io/region": "${AWS_REGION}",
                "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:RequestTag/topology.kubernetes.io/region": "${AWS_REGION}"
            },
            "StringLike": {
                "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*",
                "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
            }
            }
        },
        {
            "Sid": "AllowScopedInstanceProfileActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": [
            "iam:AddRoleToInstanceProfile",
            "iam:RemoveRoleFromInstanceProfile",
            "iam:DeleteInstanceProfile"
            ],
            "Condition": {
            "StringEquals": {
                "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                "aws:ResourceTag/topology.kubernetes.io/region": "${AWS_REGION}"
            },
            "StringLike": {
                "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*"
            }
            }
        },
        {
            "Sid": "AllowInstanceProfileReadActions",
            "Effect": "Allow",
            "Resource": "*",
            "Action": "iam:GetInstanceProfile"
        }
    ],
    "Version": "2012-10-17"
}
EOF

```

Attach custom inline policy:

```bash
aws iam put-role-policy \
  --role-name "KarpenterControllerRole-${CLUSTER_NAME}" \
  --policy-name "KarpenterControllerPolicy-${CLUSTER_NAME}" \
  --policy-document file://controller-policy.json
```

Make sure you’ve created the `controller-policy.json` as shown in the original script (skip here for brevity).

---

## 🏷️ Step 4: Tag Subnets and Security Groups

Karpenter needs to discover networking resources.

```bash
for NODEGROUP in $(aws eks list-nodegroups --cluster-name "${CLUSTER_NAME}" --query 'nodegroups' --output text); do
  aws ec2 create-tags \
    --resources $(aws eks describe-nodegroup --cluster-name "${CLUSTER_NAME}" --nodegroup-name "${NODEGROUP}" --query 'nodegroup.subnets' --output text) \
    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}"
done
```

**Security Groups:**

Tag all SGs used by EKS with:

```bash
Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}
```

---

## 🚀 Step 5: Update aws-auth ConfigMap

```bash
kubectl edit configmap aws-auth -n kube-system
```
```bash
- groups:
  - system:bootstrappers
  - system:nodes
  ## If you intend to run Windows workloads, the kube-proxy group should be specified.
  # For more information, see https://github.com/aws/karpenter/issues/5099.
  # - eks:kube-proxy-windows
  rolearn: arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}
  username: system:node:{{EC2PrivateDNSName}}

```

---

## 🚀 Step 6: Deploy Karpenter with Helm

```bash
aws ecr-public get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin public.ecr.aws

https://karpenter.sh/docs/getting-started/migrating-from-cas/

https://github.com/aws/karpenter-provider-aws/releases

export KARPENTER_VERSION="1.4.0"

helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi
```

---

## 🧱 Step 7: Deploy a NodePool

Example for ARM-based (Graviton) workloads:

```yaml
# NodePool to launch Graviton (arm64) instances
https://github.com/aws/karpenter-provider-aws/blob/main/examples/v1/general-purpose.yaml

https://github.com/aws/karpenter-provider-aws/tree/main/examples/v1
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose-arm
  annotations:
    kubernetes.io/description: "Graviton NodePool for ARM-based workloads"
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["arm64"]  # ✅ ARM architecture
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]  # Includes Graviton families like c7g, m7g, r7g
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["6"]  # Graviton2 (generation 6) or higher
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: graviton-default
---
# EC2NodeClass for Graviton (Amazon Linux 2023)
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: graviton-default
  annotations:
    kubernetes.io/description: "Graviton EC2NodeClass for ARM-based Amazon Linux 2023 nodes"
spec:
  role: "KarpenterNodeRole-Non-Prod-EduNext"  # Replace with your actual cluster name or inject via env
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "Non-Prod-EduNext"  # Replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "Non-Prod-EduNext"  # Replace with your cluster name
  amiSelectorTerms:
    - id: ami-02f67279123ee4a30  # Amazon Linux 2023 (supports arm64 & x86_64)
  amiFamily: AL2023
```

Apply the NodePool:

```bash
kubectl apply -f NodePool.yaml
```

---

## ✅ Done!

Karpenter is now installed and configured to auto-provision compute for your EKS workloads.

---

## 🔍 References

- https://karpenter.sh/docs/
- https://docs.aws.amazon.com/eks/latest/userguide/karpenter.html
