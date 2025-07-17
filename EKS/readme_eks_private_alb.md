# Deploy Web App on EKS with Internal ALB

## 📘 Project Overview

This guide explains how to deploy a simple web application on AWS EKS with:

- Dockerized app pushed to ECR
- Kubernetes deployment & service
- Private subnet-only node group
- Internal ALB (Application Load Balancer)

## 🧹 Tools Used

| Tool    | Purpose                        |
| ------- | ------------------------------ |
| Docker  | Build and containerize web app |
| ECR     | Store container image          |
| EKS     | Kubernetes cluster             |
| eksctl  | Create cluster and nodegroups  |
| kubectl | Deploy and manage K8s objects  |
| helm    | Install ALB Ingress controller |

## 📂 Directory Structure

```
eks-private-alb-webapp/
├── Dockerfile
├── index.html
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

## 🧠 Step-by-Step Deployment Guide

### 1️⃣ Create and Push Docker Image

**a. Create ECR Repo**

```bash
aws ecr create-repository --repository-name private-web-app
```

**b. Authenticate Docker to ECR**

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin <your_account_id>.dkr.ecr.<region>.amazonaws.com
```

**c. Build and Push Image**

```bash
docker build -t private-web-app .
docker tag private-web-app:latest <your_account_id>.dkr.ecr.<region>.amazonaws.com/private-web-app:latest
docker push <your_account_id>.dkr.ecr.<region>.amazonaws.com/private-web-app:latest
```

### 2️⃣ Create EKS Cluster (No Nodes)

```bash
eksctl create cluster \
  --name private-alb-cluster \
  --region <region> \
  --vpc-public-subnets <public-subnet-ids> \
  --vpc-private-subnets <private-subnet-ids> \
  --without-nodegroup
```

### 3️⃣ Create Nodegroup in Private Subnet

```bash
eksctl create nodegroup \
  --cluster private-alb-cluster \
  --region <region> \
  --name private-node-group \
  --node-type t3.medium \
  --nodes 2 \
  --node-private-networking \
  --subnet-ids <private-subnet-ids>
```

### 4️⃣ Set Up ALB Ingress Controller

**a. OIDC Setup**

```bash
eksctl utils associate-iam-oidc-provider \
  --region <region> \
  --cluster private-alb-cluster \
  --approve
```

**b. IAM Policy for ALB Controller**

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

**c. Create Service Account for ALB Controller**

```bash
eksctl create iamserviceaccount \
  --cluster private-alb-cluster \
  --region <region> \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<your_account_id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**d. Install ALB Controller via Helm**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=private-alb-cluster \
  --set serviceAccount.create=false \
  --set region=<region> \
  --set vpcId=<vpc-id> \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set ingressClass=alb
```

### 5️⃣ Deploy App (Kubernetes)

**a. Update **``** with ECR Image**

```yaml
image: <your-account-id>.dkr.ecr.<region>.amazonaws.com/private-web-app:latest
```

**b. Update **``** with Private Subnet IDs**

```yaml
alb.ingress.kubernetes.io/subnets: subnet-abc123,subnet-def456
```

**c. Apply YAMLs**

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

## ✅ Result

| Resource | Behavior                             |
| -------- | ------------------------------------ |
| ALB      | Created in private subnet            |
| Web App  | Hosted on EKS nodes (private subnet) |
| DNS Name | Internal-only ALB                    |
| Access   | From EC2/Bastion/VPN inside VPC      |

## 🧪 Test

```bash
kubectl get ingress
kubectl describe ingress web-app-ingress
```

SSH into a bastion EC2 or use SSM Session Manager in same VPC:

```bash
curl http://<ALB-DNS-name>
```

## ❓ FAQ

**Q: Why not manually create ALB?**

> Because Kubernetes Ingress + ALB Controller manages it dynamically, following K8s best practices.

**Q: Can I make it public later?**

> Yes! Change this in ingress:

```yaml
alb.ingress.kubernetes.io/scheme: internet-facing
```

