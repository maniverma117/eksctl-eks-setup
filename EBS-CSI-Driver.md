
# 🚀 Amazon EBS CSI Driver Setup for EKS Cluster

This guide helps you install the **Amazon EBS CSI Driver** on your EKS cluster and validate it using a sample `StorageClass`, `PVC`, and Pod.


### 📍 Prerequisites

* `eksctl` installed
* `aws` CLI configured with admin access
* EKS cluster named `Prod-EduNext` in `ap-south-1`
* Kubernetes version 1.32
* IAM OIDC provider not already associated

---

### 🪪 1. Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region=ap-south-1 \
  --cluster=Prod-EduNext \
  --approve

aws eks describe-cluster --name "Prod-EduNext" --query "cluster.identity.oidc.issuer" --output text 
```

---

### 🔐 2. Create Trust Policy File

Create a file called `trust-policy.json` with the following content:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::927808981943:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/47D832B0899E9015E3EFB34BF7494E12"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/47D832B0899E9015E3EFB34BF7494E12:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
```

---

### 📄 3. (Optional) Create Custom IAM Policy (if not using AWS-managed one)

Create a file named `AmazonEBSCSIDriverPolicy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:CreateVolume",
        "ec2:DeleteVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

Then run:

```bash
aws iam create-policy \
  --policy-name AmazonEBSCSIDriverPolicy \
  --policy-document file://AmazonEBSCSIDriverPolicy.json
```

Or use the AWS-managed policy:

```text
arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

---

### 🧰 4. Create IAM Role and Service Account

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster Prod-EduNext \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

Update the trust relationship:

```bash
aws iam update-assume-role-policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-document file://trust-policy.json
```

Then bind the role to the service account:

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster Prod-EduNext \
  --region ap-south-1 \
  --attach-role-arn arn:aws:iam::927808981943:role/AmazonEKS_EBS_CSI_DriverRole \
  --approve
```

---

### 🧩 5. Check Addon Versions (optional)

```bash
eksctl utils describe-addon-versions \
  --kubernetes-version 1.32 \
  --region ap-south-1 | grep AddonName

eksctl utils describe-addon-versions \
  --kubernetes-version 1.32 \
  --name aws-ebs-csi-driver \
  --region ap-south-1 | grep AddonVersion
```

---

### ➕ 6. Install EBS CSI Driver Addon

```bash
eksctl create addon \
  --cluster Prod-EduNext \
  --name aws-ebs-csi-driver \
  --version latest \
  --region ap-south-1 \
  --service-account-role-arn arn:aws:iam::927808981943:role/AmazonEKS_EBS_CSI_DriverRole
```

---

### 📦 7. Create StorageClass

Save the following as `EBS-backed-StorageClass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Apply it:

```bash
kubectl apply -f EBS-backed-StorageClass.yaml
```

---

### 🧪 8. Test Persistent Volume Claim

Create the following file as `test-claim.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: public.ecr.aws/amazonlinux/amazonlinux
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-claim
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
```

Apply it:

```bash
kubectl apply -f test-claim.yaml
```

Verify it’s working:

```bash
kubectl get pvc
kubectl get pods
kubectl logs app
```

You should see timestamps being written to `/data/out.txt` on the EBS volume.

---

✅ **Your Amazon EBS CSI driver setup is now complete and validated on EKS!**

