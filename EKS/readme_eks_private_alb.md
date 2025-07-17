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
---------------------------------------------------------------------------------------

# 🛠 Manual ALB Setup for EKS Web App (Private/Internal)

This guide provides step-by-step instructions for manually configuring an **Application Load Balancer (ALB)** in **private/internal mode** for an application running on an **Amazon EKS cluster**.

---

## 📌 Assumptions

- You already have a running EKS cluster.
- Your application is exposed via a **Kubernetes Service of type **``.
- You want the ALB to be internal/private (accessible only inside the VPC).

---

## 📁 Project Structure

```
eks-private-alb-webapp/
├── service.yaml # Kubernetes Service manifest
├── README.md    # This file
```

---

## 1️⃣ Create Kubernetes NodePort Service

If you haven't already, define a NodePort service for your app:

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080  # Use custom or let K8s allocate one
```

Apply the service:

```bash
kubectl apply -f service.yaml
```

---

## 2️⃣ Create Target Group in AWS

```bash
aws elbv2 create-target-group \
  --name web-app-targets \
  --protocol HTTP \
  --port 30080 \
  --vpc-id <your-vpc-id> \
  --target-type instance
```

♻️ Use `--target-type ip` if using custom networking (ENI mode) or if your EKS nodes are in a different account.

---

## 3️⃣ Register EKS Worker Nodes

Get instance IDs of your worker nodes:

```bash
aws ec2 describe-instances \
  --filters Name=tag:eks:cluster-name,Values=<your-cluster-name>
```

Register the instances:

```bash
aws elbv2 register-targets \
  --target-group-arn <target-group-arn> \
  --targets Id=i-xxxxxx,Port=30080
```

---

## 4️⃣ Create Internal ALB

```bash
aws elbv2 create-load-balancer \
  --name web-app-internal-alb \
  --subnets subnet-xxxx subnet-yyyy \
  --scheme internal \
  --type application \
  --security-groups sg-xxxxxx
```

---

## 5️⃣ Create HTTP Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<target-group-arn>
```

---

## 6️⃣ Test Access Internally

From an EC2 instance or bastion host in the same VPC:

```bash
curl http://<internal-alb-dns-name>
```

---

## ✅ Summary

| Resource      | Type     | Notes                                   |
| ------------- | -------- | --------------------------------------- |
| Load Balancer | Internal | Manually created in VPC                 |
| Target Group  | HTTP     | Points to EKS worker nodes via NodePort |
| Service       | NodePort | Exposes the app for ALB                 |
| Access        | Private  | Internal-only (within VPC)              |

---

## 🔐 Security Tips

- Ensure the security group for the ALB allows inbound traffic on port 80 only from trusted sources.
- Use NACLs and security group rules to restrict access further if needed.
- Monitor ALB and target group health status via CloudWatch or ELB console.

---

## 🛋️ Cleanup Resources

To avoid unexpected charges:

```bash
# Delete listener
aws elbv2 delete-listener --listener-arn <listener-arn>

# Delete load balancer
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn>

# Deregister and delete target group
aws elbv2 deregister-targets --target-group-arn <target-group-arn> --targets Id=i-xxxxxx
aws elbv2 delete-target-group --target-group-arn <target-group-arn>

# Delete Kubernetes service
kubectl delete -f service.yaml
```