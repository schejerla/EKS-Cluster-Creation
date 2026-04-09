# 🚀 EKS Cluster Creation

A shell-script-based guide and automation for provisioning an Amazon EKS (Elastic Kubernetes Service) cluster using `eksctl`, along with node group setup, OIDC configuration, and `kubectl` integration.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Installation](#installation)
- [Usage](#usage)
- [Cluster Configuration](#cluster-configuration)
- [Post-Setup Validation](#post-setup-validation)
- [Teardown / Cleanup](#teardown--cleanup)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

This project automates the creation of a production-ready Amazon EKS cluster using `eksctl`. It provisions the control plane, managed node groups, OIDC provider, and kubeconfig — all from a single script or set of commands.

---

## ✅ Prerequisites

Before running any scripts in this repository, ensure the following tools and permissions are in place.

---

### 1. 🖥️ Operating System

- Linux (Amazon Linux 2, Ubuntu 20.04+) or macOS
- Windows users should use **WSL2** or **Git Bash**

---

### 2. 🔑 AWS Account & IAM Permissions

You need an active AWS account with an IAM user or role that has the following permissions:

| Permission Area | Why It's Needed |
|---|---|
| `AmazonEKSClusterPolicy` | Create/manage EKS control plane |
| `AmazonEKSWorkerNodePolicy` | Manage EC2 worker nodes |
| `AmazonEC2FullAccess` | Provision VPC, subnets, security groups |
| `IAMFullAccess` | Create roles, OIDC providers, service accounts |
| `AWSCloudFormationFullAccess` | eksctl uses CloudFormation stacks internally |
| `AmazonVPCFullAccess` | Create/manage VPC and networking |

> **Tip:** For quick setup, you can use `AdministratorAccess` on a dev/sandbox account. For production, use least-privilege scoped policies.

---

### 3. 🛠️ Required CLI Tools

Install and configure all of the following tools before proceeding:

#### AWS CLI v2

```bash
# Install (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
# Expected: aws-cli/2.x.x

# Configure credentials
aws configure
# Enter: AWS Access Key ID, Secret Access Key, Region, Output format (json)
```

#### eksctl

```bash
# Install (Linux/macOS)
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verify
eksctl version
# Expected: 0.170.x or higher
```

#### kubectl

```bash
# Install latest stable (Linux)
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

#### Helm (optional, for add-ons)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

### 4. 🔐 SSH Key Pair (optional but recommended)

If you plan to SSH into worker nodes:

```bash
# Create key pair via AWS CLI
aws ec2 create-key-pair \
  --key-name eks-keypair \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/eks-keypair.pem

chmod 400 ~/.ssh/eks-keypair.pem
```

Or use an existing key pair — note the name for the eksctl command.

---

### 5. 🌐 Networking Requirements

Ensure the target AWS region has:

- At least **2 Availability Zones**
- Either the default VPC (eksctl can auto-create) or a pre-existing VPC with:
  - Public and/or private subnets across ≥2 AZs
  - Internet Gateway (for public subnets)
  - NAT Gateway (for private subnets)
  - DNS hostnames and DNS resolution **enabled**

---

### 6. 📦 Software Version Reference

| Tool | Minimum Version |
|---|---|
| AWS CLI | v2.0+ |
| eksctl | v0.150+ |
| kubectl | Matches EKS cluster version (±1 minor) |
| Helm | v3.x (optional) |
| Kubernetes | 1.27+ (recommended) |

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────┐
│                  AWS Cloud                   │
│                                              │
│   ┌──────────────────────────────────────┐   │
│   │           VPC (10.0.0.0/16)          │   │
│   │                                      │   │
│   │  ┌───────────┐   ┌───────────┐      │   │
│   │  │ Public    │   │ Private   │      │   │
│   │  │ Subnet AZ1│   │ Subnet AZ1│      │   │
│   │  └───────────┘   └───────────┘      │   │
│   │  ┌───────────┐   ┌───────────┐      │   │
│   │  │ Public    │   │ Private   │      │   │
│   │  │ Subnet AZ2│   │ Subnet AZ2│      │   │
│   │  └───────────┘   └───────────┘      │   │
│   │                                      │   │
│   │  ┌──────────────────────────────┐   │   │
│   │  │      EKS Control Plane       │   │   │
│   │  │   (AWS Managed / HA)         │   │   │
│   │  └──────────────────────────────┘   │   │
│   │                                      │   │
│   │  ┌──────────────────────────────┐   │   │
│   │  │  Managed Node Group          │   │   │
│   │  │  EC2 Worker Nodes (t3.medium)│   │   │
│   │  └──────────────────────────────┘   │   │
│   └──────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

---

## 🚀 Installation

```bash
# Clone this repository
git clone https://github.com/schejerla/EKS-Cluster-Creation.git
cd EKS-Cluster-Creation

# Make the script executable
chmod +x EKS-Cluster-Creation
```

---

## 🔧 Usage

### Step 1 — Create EKS Cluster (without node group)

```bash
eksctl create cluster \
  --name=eksdemo \
  --region=us-east-1 \
  --zones=us-east-1a,us-east-1b \
  --without-nodegroup
```

### Step 2 — Associate OIDC Provider

Required for IAM Roles for Service Accounts (IRSA):

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster eksdemo \
  --approve
```

### Step 3 — Create Managed Node Group

```bash
eksctl create nodegroup \
  --cluster=eksdemo \
  --region=us-east-1 \
  --name=eksdemo-ng-public \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=4 \
  --node-volume-size=20 \
  --ssh-access \
  --ssh-public-key=eks-keypair \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access
```

### Step 4 — Update kubeconfig

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name eksdemo
```

---

## ⚙️ Cluster Configuration

You can also use a YAML config file instead of CLI flags:

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksdemo
  region: us-east-1
  version: "1.29"

managedNodeGroups:
  - name: ng-public
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    volumeSize: 20
    ssh:
      allow: true
      publicKeyName: eks-keypair
    iam:
      withAddonPolicies:
        albIngress: true
        autoScaler: true
        ebs: true
        externalDNS: true
```

```bash
eksctl create cluster -f cluster-config.yaml
```

---

## ✔️ Post-Setup Validation

```bash
# Check cluster info
kubectl cluster-info

# List all nodes
kubectl get nodes -o wide

# List all pods (system namespace)
kubectl get pods -n kube-system

# Verify CloudFormation stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --region us-east-1
```

---

## 🗑️ Teardown / Cleanup

```bash
# Delete node group first
eksctl delete nodegroup \
  --cluster=eksdemo \
  --region=us-east-1 \
  --name=eksdemo-ng-public

# Delete the cluster
eksctl delete cluster \
  --name=eksdemo \
  --region=us-east-1
```

> ⚠️ This will delete all associated CloudFormation stacks, VPCs, and node groups. Ensure you have backups of any persistent data before running cleanup.

---

## 🛠️ Troubleshooting

| Issue | Solution |
|---|---|
| `aws: command not found` | Reinstall AWS CLI v2 and check `$PATH` |
| `eksctl: command not found` | Move binary to `/usr/local/bin` or update `$PATH` |
| IAM permission denied | Attach required IAM policies to your user/role |
| Cluster creation times out | Check CloudFormation events in the AWS console |
| Nodes in `NotReady` state | Verify node group IAM role has `AmazonEKSWorkerNodePolicy` |
| kubectl can't connect | Re-run `aws eks update-kubeconfig` with correct region/cluster name |

---

## 📚 References

- [Amazon EKS Official Docs](https://docs.aws.amazon.com/eks/latest/userguide/)
- [eksctl GitHub](https://github.com/weaveworks/eksctl)
- [kubectl Install Guide](https://kubernetes.io/docs/tasks/tools/)
- [AWS CLI Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)

