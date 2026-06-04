# 🎮 Deploying 2048 Game on AWS EKS with Fargate & ALB Ingress Controller

[![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/eks/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Fargate](https://img.shields.io/badge/AWS-Fargate-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/fargate/)
[![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-VM-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)

---

## 📌 Project Overview

This project demonstrates a **production-grade Kubernetes deployment** of the classic 2048 game on **Amazon EKS (Elastic Kubernetes Service)** using **AWS Fargate** (serverless compute) and the **AWS Load Balancer Controller** for public ingress. The entire infrastructure is provisioned and managed using `eksctl`, `kubectl`, and `Helm` — all executed from an **Ubuntu Linux VM** as the local control environment.

> **Goal:** Deploy a containerized web application to a fully managed EKS cluster with a public-facing Application Load Balancer (ALB) — no EC2 node management required.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────┐
│   Local Environment                 │
│   Ubuntu VM                         │
│   ├── AWS CLI (aws configure)       │
│   ├── eksctl (cluster management)   │
│   ├── kubectl (workload control)    │
│   └── Helm  (package manager)       │
└──────────────┬──────────────────────┘
               │  eksctl / kubectl / helm
               ▼
          AWS Cloud
               │
               ▼
┌─────────────────────────────┐
│  Application Load Balancer  │  ← AWS ALB (via Ingress)
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│     EKS Cluster             │
│  ┌───────────────────────┐  │
│  │  Namespace: game-2048 │  │
│  │  ┌─────────────────┐  │  │
│  │  │  2048 Pods       │  │  │  ← Fargate (Serverless)
│  │  │  (Deployment)    │  │  │
│  │  └─────────────────┘  │  │
│  │  Service (NodePort)    │  │
│  │  Ingress Resource      │  │
│  └───────────────────────┘  │
│                              │
│  ┌───────────────────────┐  │
│  │  Namespace: kube-system│  │
│  │  ALB Controller Pod    │  │  ← Manages ALB lifecycle
│  └───────────────────────┘  │
└─────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Tool / Service | Purpose |
|---|---|
| **Ubuntu VM** | Local control environment to run all CLI tools |
| **Amazon EKS** | Managed Kubernetes control plane |
| **AWS Fargate** | Serverless compute for pods (no EC2 management) |
| **eksctl** | CLI to provision EKS cluster and Fargate profiles |
| **kubectl** | Kubernetes CLI to manage workloads |
| **AWS ALB (Ingress)** | Expose the app publicly via Application Load Balancer |
| **AWS Load Balancer Controller** | Kubernetes controller to manage ALB lifecycle |
| **Helm** | Package manager to install ALB Controller |
| **IAM OIDC Provider** | Enables pods to securely assume IAM roles |
| **IAM Service Account** | Fine-grained AWS permissions for ALB controller pod |

---

## 📋 Prerequisites

Before you begin, ensure you have:

- ✅ **Ubuntu Linux VM** (this project was executed on Ubuntu)
- ✅ An **AWS Account** with admin/IAM permissions
- ✅ **AWS CLI** configured (`aws configure`)
- ✅ **eksctl** installed
- ✅ **kubectl** installed
- ✅ **Helm** installed
- ✅ An existing **VPC** (note your VPC ID)

---

## 🚀 Step-by-Step Deployment

### Step 1 — Install Required Tools (on Ubuntu VM)

> 🖥️ All commands below are run on an **Ubuntu Linux VM**. The `apt` package manager and `snap` are used for installations.

```bash
# Install curl and tar
sudo apt update && sudo apt install -y curl tar unzip

# Install eksctl
ARCH=$(uname -m | sed 's/x86_64/amd64/' | sed 's/aarch64/arm64/')
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
sudo tar -xzf eksctl_$PLATFORM.tar.gz -C /usr/local/bin && rm eksctl_$PLATFORM.tar.gz
eksctl version

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Install Helm
sudo snap install helm --classic
```

---

### Step 2 — Configure AWS Credentials

```bash
aws configure
# Enter: AWS Access Key ID, Secret Access Key, Region (us-east-1), Output format (json)
```

---

### Step 3 — Create EKS Cluster with Fargate

```bash
eksctl create cluster \
  --name EKS-Cluster \
  --region us-east-1 \
  --fargate
```

> ⏳ This takes ~15–20 minutes. It provisions the EKS control plane and a default Fargate profile.

```bash
# Update kubeconfig to connect kubectl to your new cluster
aws eks update-kubeconfig --name EKS-Cluster --region us-east-1
kubectl get all
```

---

### Step 4 — Create a Fargate Profile for the App Namespace

```bash
eksctl create fargateprofile \
  --cluster EKS-Cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

---

### Step 5 — Deploy the 2048 Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

# Verify deployment
kubectl get pods -n game-2048
kubectl get service -n game-2048
kubectl get deploy -n game-2048
kubectl get ingress -n game-2048
```

> At this point, the Ingress will have no ADDRESS yet — the ALB Controller is not installed yet.

---

### Step 6 — Set Up IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster EKS-Cluster \
  --approve
```

> This allows Kubernetes service accounts to assume IAM roles (required for ALB controller).

---

### Step 7 — Create IAM Policy for ALB Controller

```bash
# Download the policy document
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

# Create the IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

---

### Step 8 — Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=EKS-Cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

> 🔁 Replace `<YOUR_ACCOUNT_ID>` with your 12-digit AWS Account ID.

---

### Step 9 — Install AWS Load Balancer Controller via Helm

```bash
# Add EKS Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=EKS-Cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<YOUR_VPC_ID>
```

> 🔁 Replace `<YOUR_VPC_ID>` with your actual VPC ID (e.g., `vpc-0abc1234def56789`).

---

### Step 10 — Verify the Deployment

```bash
# Check ALB controller is running
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check all pods in kube-system
kubectl get pods -n kube-system

# Get the Ingress — should now show an ADDRESS (ALB DNS)
kubectl get ingress -n game-2048
```

Once the `ADDRESS` column shows a DNS name (e.g., `k8s-game2048-xxxx.us-east-1.elb.amazonaws.com`), paste it in your browser — **the 2048 game is live!** 🎉

---

## 📂 Project Structure

```
.
├── README.md                  # This file
└── iam_policy.json            # IAM policy for ALB Controller (auto-downloaded)
```

---

## 🧹 Cleanup (Avoid AWS Charges)

```bash
# Delete the EKS cluster and all associated resources
eksctl delete cluster --name EKS-Cluster --region us-east-1
```

> ⚠️ **Important:** Always delete the cluster after testing to avoid unexpected AWS charges.

---

## 💡 Key Concepts Demonstrated

- **Ubuntu VM as Control Node**: All AWS and Kubernetes CLI tools were installed and executed on an Ubuntu Linux VM, simulating a real DevOps workstation
- **EKS + Fargate**: Serverless Kubernetes — no EC2 worker node management
- **Kubernetes Ingress**: Declarative routing from ALB to backend service
- **IAM Roles for Service Accounts (IRSA)**: Secure, least-privilege AWS access for pods
- **Helm Charts**: Streamlined installation of complex Kubernetes controllers
- **eksctl**: Infrastructure-as-CLI for EKS cluster lifecycle management

---

## 👨‍💻 Author

**Your Name**
- GitHub: [@yourusername](https://github.com/yourusername)
- LinkedIn: [linkedin.com/in/yourprofile](https://linkedin.com/in/yourprofile)

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

> ⭐ If you found this project helpful, please give it a star!
