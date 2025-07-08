# 🌐 Deploy ExternalDNS on EKS (Helm + AWS Route53)

This guide helps you deploy to automatically manage Route53 DNS records for your EKS services and ingresses.

---

## 📦 Tooling Used

* **Helm**: to deploy the ExternalDNS chart.
* **Bitnami ExternalDNS Chart**: hosted on Docker Hub's OCI registry.
* **Route53**: as the DNS provider (public zone).
* **IAM with IRSA**: assumed to already be configured.

---

## ✅ Prerequisites

* EKS cluster with working `kubectl` and `helm` access.
* OIDC provider associated with the cluster (`eksctl utils associate-iam-oidc-provider`).
* IAM Role for ExternalDNS created and mapped to a service account with Route53 permissions (see Note below).
* Hosted Zone ID in Route53: `Z0041638LDEUL5U7QZS2`
* Public domain: `Test-project.com`

---

## 🛠️ Deployment

Run the following command to install **ExternalDNS** via Helm:

```bash
helm install edunext \
  --set provider=aws \
  --set aws.zoneType=public \
  --set txtOwnerId=Z0041638LDEUL5U7QZS2 \
  --set domainFilters[0]=test-project.com \
  oci://registry-1.docker.io/bitnamicharts/external-dns
```

---

## ⚙️ What Each Parameter Means

| Parameter                         | Description                                                       |
| --------------------------------- | ----------------------------------------------------------------- |
| `provider=aws`                    | Tells ExternalDNS to use AWS Route53 as the DNS provider          |
| `aws.zoneType=public`             | Specifies you're using public hosted zones in Route53             |
| `txtOwnerId=Z0041638LDEUL5U7QZS2` | Used to identify the records created by this ExternalDNS instance |
| `domainFilters[0]=Test-project.app`    | Limits ExternalDNS to only manage records for this domain         |

---

## 🔐 IAM Permissions Required (Optinal)

Make sure the IAM role attached to the ExternalDNS service account has the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "*"
    }
  ]
}
```

You can attach this policy to an IAM role, and associate it with the Kubernetes service account like this:

```bash
eksctl create iamserviceaccount \
  --cluster Test-project \
  --namespace kube-system \
  --name external-dns \
  --attach-policy-arn arn:aws:iam::92xxxxxxx943:policy/ExternalDNSAccessPolicy \
  --region ap-south-1 \
  --approve
```

---

## 📌 Verify Installation

Check that the ExternalDNS pod is running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=external-dns
```

---

## 🚀 Test by Creating Ingress

Create a sample ingress annotated with your domain and check if records are created in Route53.

---

## 🧼 Cleanup

To uninstall:

```bash
helm uninstall Test-project
