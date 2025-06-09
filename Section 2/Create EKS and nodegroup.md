# EKS Cluster Setup with `eksctl`

This guide outlines the steps to create an EKS cluster, associate IAM OIDC provider, set up an EC2 key pair, and create a node group with useful add-ons.

---

## 1. Create EKS Cluster (Control Plane Only)

```bash
eksctl create cluster \
  --name=eksdemo1 \
  --region=us-east-1 \
  --zones=us-east-1a,us-east-1b \
  --without-nodegroup
```

### Get List of Clusters

```bash
eksctl get clusters
```

---

## 2. Associate IAM OIDC Provider

To enable IAM roles for Kubernetes service accounts, associate an IAM OIDC provider with your EKS cluster:

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster eksdemo1 \
  --approve
```

---

## 3. Create an EC2 Key Pair

Create a new EC2 key pair named `kube-demo` (used for SSH access to worker nodes):

```bash
aws ec2 create-key-pair \
  --region us-east-1 \
  --key-name kube-demo \
  --query 'KeyMaterial' \
  --output text > kube-demo.pem

chmod 400 kube-demo.pem
```

---

## 4. Create Node Group with Add-Ons in Public Subnets

```bash
eksctl create nodegroup \
  --cluster=eksdemo1 \
  --region=us-east-1 \
  --name=eksdemo1-ng-public1 \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=4 \
  --node-volume-size=20 \
  --ssh-access \
  --ssh-public-key=kube-demo \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access
```

> 📘 Use `eksctl create --help`, `eksctl create cluster --help`, or `eksctl create nodegroup --help` for more information on each command.

---

## 5. Verify Cluster and Nodes

```bash
eksctl get cluster

eksctl get nodegroup --cluster=eksdemo1

kubectl get nodes -o wide

kubectl config view --minify
```

---

## 6. EKS clsuter pricing
 
  - EKS is not for free
  - inshort, no free tier for EKS

  - We pay $0.10 per hour for each amazon EKS cluster
  - per day: $2.4s
  - For 30days: $72 

---

## ✅ Summary

You’ve now:
- Created an EKS cluster control plane.
- Associated an OIDC provider for IAM roles.
- Created a key pair for SSH access.
- Deployed a managed node group with helpful AWS add-ons.
- Verified your setup using `eksctl` and `kubectl`.
