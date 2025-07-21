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

** log
```bash
2025-07-21 17:18:27 [ℹ]  eksctl version 0.207.0
2025-07-21 17:18:27 [ℹ]  using region us-east-1
2025-07-21 17:18:28 [✔]  using existing VPC (vpc-045a67b30a03f42d0) and subnets (private:map[us-east-1a:{subnet-0e9417d9895394eb6 us-east-1a 192.168.2.0/24 0 } us-east-1b:{subnet-0905412cf0ba9a59d us-east-1b 192.168.3.0/24 0 }] public:map[])
2025-07-21 17:18:28 [!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
2025-07-21 17:18:28 [ℹ]  using Kubernetes version 1.32
2025-07-21 17:18:28 [ℹ]  creating EKS cluster "private-alb-cluster01" in "us-east-1" region with
2025-07-21 17:18:28 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-east-1 --cluster=private-alb-cluster01'
2025-07-21 17:18:28 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "private-alb-cluster01" in "us-east-1"
2025-07-21 17:18:28 [ℹ]  CloudWatch logging will not be enabled for cluster "private-alb-cluster01" in "us-east-1"
2025-07-21 17:18:28 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=us-east-1 --cluster=private-alb-cluster01'
2025-07-21 17:18:28 [ℹ]  default addons metrics-server, vpc-cni, kube-proxy, coredns were not specified, will install them as EKS addons
2025-07-21 17:18:28 [ℹ]
2 sequential tasks: { create cluster control plane "private-alb-cluster01",
    2 sequential sub-tasks: {
        1 task: { create addons },
        wait for control plane to become ready,
    }
}
2025-07-21 17:18:28 [ℹ]  building cluster stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:18:28 [ℹ]  deploying stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:18:58 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:19:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:20:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:21:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:22:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:23:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:24:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:25:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:26:28 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:27:29 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-cluster"
2025-07-21 17:27:29 [ℹ]  creating addon: metrics-server
2025-07-21 17:27:30 [ℹ]  successfully created addon: metrics-server
2025-07-21 17:27:30 [!]  recommended policies were found for "vpc-cni" addon, but since OIDC is disabled on the cluster, eksctl cannot configure the requested permissions; the recommended way to provide IAM permissions for "vpc-cni" addon is via pod identity associations; after addon creation is completed, add all recommended policies to the config file, under `addon.PodIdentityAssociations`, and run `eksctl update addon`
2025-07-21 17:27:30 [ℹ]  creating addon: vpc-cni
2025-07-21 17:27:30 [ℹ]  successfully created addon: vpc-cni
2025-07-21 17:27:31 [ℹ]  creating addon: kube-proxy
2025-07-21 17:27:31 [ℹ]  successfully created addon: kube-proxy
2025-07-21 17:27:31 [ℹ]  creating addon: coredns
2025-07-21 17:27:31 [ℹ]  successfully created addon: coredns
2025-07-21 17:29:32 [ℹ]  waiting for the control plane to become ready
2025-07-21 17:29:32 [✔]  saved kubeconfig as "/root/.kube/config"
2025-07-21 17:29:32 [ℹ]  no tasks
2025-07-21 17:29:32 [✔]  all EKS cluster resources for "private-alb-cluster01" have been created
2025-07-21 17:29:33 [ℹ]  kubectl command should work with "/root/.kube/config", try 'kubectl get nodes'
2025-07-21 17:29:33 [✔]  EKS cluster "private-alb-cluster01" in "us-east-1" region is ready
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

** log
```bash
2025-07-21 17:31:08 [ℹ]  will use version 1.32 for new nodegroup(s) based on control plane version
2025-07-21 17:31:09 [ℹ]  nodegroup "private-node-group01" will use "" [AmazonLinux2/1.32]
2025-07-21 17:31:09 [ℹ]  1 nodegroup (private-node-group01) was included (based on the include/exclude rules)
2025-07-21 17:31:09 [ℹ]  will create a CloudFormation stack for each of 1 managed nodegroups in cluster "private-alb-cluster01"2025-07-21 17:31:09 [ℹ]
2 sequential tasks: { fix cluster compatibility, 1 task: { 1 task: { create managed nodegroup "private-node-group01" } }
}
2025-07-21 17:31:09 [ℹ]  checking cluster stack for missing resources
2025-07-21 17:31:09 [ℹ]  cluster stack has all required resources
2025-07-21 17:31:09 [ℹ]  building managed nodegroup stack "eksctl-private-alb-cluster01-nodegroup-private-node-group01"
2025-07-21 17:31:10 [ℹ]  deploying stack "eksctl-private-alb-cluster01-nodegroup-private-node-group01"
2025-07-21 17:31:10 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-nodegroup-private-node-group01"
2025-07-21 17:31:40 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-nodegroup-private-node-group01"
2025-07-21 17:32:33 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-nodegroup-private-node-group01"
2025-07-21 17:34:11 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-nodegroup-private-node-group01"
2025-07-21 17:34:11 [ℹ]  no tasks
2025-07-21 17:34:11 [✔]  created 0 nodegroup(s) in cluster "private-alb-cluster01"
2025-07-21 17:34:11 [ℹ]  nodegroup "private-node-group01" has 2 node(s)
2025-07-21 17:34:11 [ℹ]  node "ip-192-168-2-195.ec2.internal" is ready
2025-07-21 17:34:11 [ℹ]  node "ip-192-168-3-192.ec2.internal" is ready
2025-07-21 17:34:11 [ℹ]  waiting for at least 2 node(s) to become ready in "private-node-group01"
2025-07-21 17:34:11 [ℹ]  nodegroup "private-node-group01" has 2 node(s)
2025-07-21 17:34:11 [ℹ]  node "ip-192-168-2-195.ec2.internal" is ready
2025-07-21 17:34:11 [ℹ]  node "ip-192-168-3-192.ec2.internal" is ready
2025-07-21 17:34:11 [✔]  created 1 managed nodegroup(s) in cluster "private-alb-cluster01"
2025-07-21 17:34:11 [ℹ]  checking security group configuration for all nodegroups
2025-07-21 17:34:11 [ℹ]  all nodegroups have up-to-date cloudformation templates
```

### 4️⃣ Set Up ALB Ingress Controller

**a. OIDC Setup**

```bash
eksctl utils associate-iam-oidc-provider \
  --region <region> \
  --cluster private-alb-cluster \
  --approve
```
** log
```bash
2025-07-21 17:35:36 [ℹ]  will create IAM Open ID Connect provider for cluster "private-alb-cluster01" in "us-east-1"
2025-07-21 17:35:36 [✔]  created IAM Open ID Connect provider for cluster "private-alb-cluster01" in "us-east-1"
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

** log
```bash
2025-07-21 17:40:19 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
2025-07-21 17:40:19 [!]  serviceaccounts that exist in Kubernetes will be excluded, use --override-existing-serviceaccounts to override
2025-07-21 17:40:19 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "kube-system/aws-load-balancer-controller",
        create serviceaccount "kube-system/aws-load-balancer-controller",
    } }2025-07-21 17:40:19 [ℹ]  building iamserviceaccount stack "eksctl-private-alb-cluster01-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-21 17:40:20 [ℹ]  deploying stack "eksctl-private-alb-cluster01-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-21 17:40:20 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-21 17:40:50 [ℹ]  waiting for CloudFormation stack "eksctl-private-alb-cluster01-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-21 17:40:50 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
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

** log
```bash
NAME: aws-load-balancer-controller
LAST DEPLOYED: Mon Jul 21 17:46:55 2025
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!
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