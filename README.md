# Amazon EKS Cluster Provisioning, Workload Deployment & LoadBalancer Integration (Day 28)

A production-ready reference guide for provisioning an Amazon Elastic Kubernetes Service (EKS) cluster on AWS, configuring local `kubectl` CLI authentication, managing EC2 worker node groups, deploying scalable containerized workloads, exposing services externally via AWS Elastic Load Balancers (ELB), implementing IAM Roles for Service Accounts (IRSA), and executing automated resource teardown.

---

## Table of Contents
- [Project Architecture & Services Used](#project-architecture--services-used)
- [Prerequisites & Setup](#prerequisites--setup)
- [Step-by-Step Implementation Guide](#step-by-step-implementation-guide)
  - [Step 1: Configure IAM Roles & IRSA Permissions](#step-1-configure-iam-roles--irsa-permissions)
  - [Step 2: Provision EKS Control Plane & EC2 Node Group](#step-2-provision-eks-control-plane--ec2-node-group)
  - [Step 3: Configure `kubectl` CLI Context and Verify Cluster Nodes](#step-3-configure-kubectl-cli-context-and-verify-cluster-nodes)
  - [Step 4: Deploy Scalable Nginx Workload Manifest](#step-4-deploy-scalable-nginx-workload-manifest)
  - [Step 5: Expose Application via LoadBalancer Service & Verify Traffic](#step-5-expose-application-via-loadbalancer-service--verify-traffic)
  - [Step 6: Automated Resource Teardown & Cleanup](#step-6-automated-resource-teardown--cleanup)
- [Quick Reference Command Cheat Sheet](#quick-reference-command-cheat-sheet)
- [Troubleshooting & Common Challenges](#troubleshooting--common-challenges)
- [Security Best Practices & Hardening](#security-best-practices--hardening)
- [Learning Outcomes & Conclusion](#learning-outcomes--conclusion)

---

## Project Architecture & Services Used

```
+------------------+         +---------------------------------------+
|  You / kubectl   | ------->| EKS Control Plane (AWS-Managed)       |
+------------------+         +---------------------------------------+
                                                 |
                                                 v
+------------------+         +---------------------------------------+
|  Users / Web     | ------->| AWS LoadBalancer Service              |
+------------------+         +---------------------------------------+
                                                 |
                                                 v
                             +---------------------------------------+
                             | Managed EC2 Worker Node Group         |
                             |  ├── Nginx Pod (Replica 1)            |
                             |  └── Nginx Pod (Replica 2)            |
                             +---------------------------------------+
```

### Infrastructure Scope

| Service / Tool | Purpose | AWS / Infrastructure Scope |
| :--- | :--- | :--- |
| **Amazon EKS** | Managed Kubernetes Control Plane | Cluster `devops-winter-arc-eks` running Kubernetes `v1.31` |
| **EC2 Node Group** | Compute instances hosting container Pods | Managed EC2 worker node group across multi-AZ subnets |
| **AWS ELB** | Routing external HTTP traffic to Kubernetes Pods | Classic / Application Load Balancer created via `LoadBalancer` Service |
| **`kubectl`** | Command-line tool for Kubernetes cluster operations | Authentication context pointed to AWS EKS cluster endpoint |
| **IAM & IRSA** | Scoped security & role-based access management | IAM Roles for Service Accounts providing least-privilege pod access |
| **AWS CLI v2** | Command-line management for AWS resource lifecycle | Cluster provisioning, `kubeconfig` updating, and teardown operations |

---

## Prerequisites & Setup

- **AWS Identity & Access Management (IAM):** Active AWS account with permissions for EKS, EC2, VPC, IAM, and ELB.
- **AWS CLI v2:** Installed and configured for region `ap-south-1`.
- **`kubectl` CLI:** Installed locally to manage Kubernetes cluster objects.
- **Local Workstation Context:** User environment operating under `C:\Users\pooja`.
- **Target VPC Baseline:** Subnets tagged for load balancer auto-discovery (`kubernetes.io/role/elb = 1`).

---

## Step-by-Step Implementation Guide

### Step 1: Configure IAM Roles & IRSA Permissions

**What was done:**  
Established cluster authentication access and configured IAM Roles for Service Accounts (IRSA) to grant fine-grained permissions to Kubernetes Pods without over-privileging EC2 worker nodes.

**Commands executed:**
```powershell
# 1. Create OIDC Provider for the cluster (IRSA prerequisite)
eksctl utils associate-iam-oidc-provider `
    --cluster devops-winter-arc-eks `
    --approve `
    --region ap-south-1

# 2. Create IAM Role bound to a Kubernetes Service Account
eksctl create iamserviceaccount `
    --name nginx-sa `
    --namespace default `
    --cluster devops-winter-arc-eks `
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess `
    --approve `
    --region ap-south-1
```

**Purpose / Why:**  
IRSA decouples AWS permissions from the underlying EC2 node host. Pods adopt temporary credentials via OpenID Connect (OIDC), ensuring strict adherence to the principle of least privilege.

---

### Step 2: Provision EKS Control Plane & EC2 Node Group

**What was done:**  
Provisioned an AWS-managed Kubernetes control plane named `devops-winter-arc-eks` alongside a managed EC2 node group in region `ap-south-1`.

**Commands executed:**
```powershell
# Create EKS Cluster with Managed Node Group using eksctl
eksctl create cluster `
    --name devops-winter-arc-eks `
    --version 1.31 `
    --region ap-south-1 `
    --nodegroup-name devops-nodes `
    --node-type t3.medium `
    --nodes 2 `
    --nodes-min 1 `
    --nodes-max 3 `
    --managed
```

**Purpose / Why:**  
AWS EKS automates high availability for the Kubernetes API server and `etcd` backend. Managed node groups handle automated node provisioning, patching, and lifecycle updates.

---

### Step 3: Configure `kubectl` CLI Context and Verify Cluster Nodes

**What was done:**  
Updated local `kubeconfig` settings using the AWS CLI, verified local active context, and inspected the node health status.

**Commands executed:**
```powershell
# 1. Update local kubeconfig context
PS C:\Users\pooja> aws eks update-kubeconfig --region ap-south-1 --name devops-winter-arc-eks

# 2. Verify current context
PS C:\Users\pooja> kubectl config current-context
arn:aws:eks:ap-south-1:************:cluster/devops-winter-arc-eks

# 3. Retrieve node status
PS C:\Users\pooja> kubectl get nodes
NAME                                         STATUS   ROLES    AGE   VERSION
ip-192-168-1-76.ap-south-1.compute.internal  Ready    <none>   12m   v1.31.0-eks-***
ip-192-168-2-19.ap-south-1.compute.internal  Ready    <none>   12m   v1.31.0-eks-***
```

**Purpose / Why:**  
Generates a valid STS token within `~/.kube/config`, allowing local commands to securely communicate with the remote AWS-managed Kubernetes API server.

---

### Step 4: Deploy Scalable Nginx Workload Manifest

**What was done:**  
Created and applied a Kubernetes `deployment.yaml` manifest specifying 2 replicas of an Nginx web application container running on port `80`.

**Manifest file (`deployment.yaml`):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
spec:
  type: LoadBalancer
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
```

**Commands executed:**
```powershell
# 1. Apply deployment manifest
PS C:\Users\pooja> kubectl apply -f deployment.yaml

# 2. Verify active running Pods
PS C:\Users\pooja> kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
nginx-demo-6d7c8f6d45-jk2sd   1/1     Running   0          3m12s
nginx-demo-6d7c8f6d45-mp9xz   1/1     Running   0          3m11s
```

**Purpose / Why:**  
Declarative manifests define the desired cluster state. Kubernetes controllers ensure 2 identical Pod replicas are scheduled and maintained across separate EC2 nodes.

---

### Step 5: Expose Application via LoadBalancer Service & Verify Traffic

**What was done:**  
Exposed the `nginx-demo` deployment using a Kubernetes `LoadBalancer` Service type, triggering AWS to automatically provision an Elastic Load Balancer (ELB) routing public web requests to the Pods.

**Commands executed:**
```powershell
# Retrieve external LoadBalancer IP / DNS endpoint
PS C:\Users\pooja> kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP                              PORT(S)        AGE
nginx-demo   LoadBalancer   10.100.12.34   a1b2c3d4e5f6g.elb.amazonaws.com          80:31612/TCP   3m
```

**Verification:**  
Navigated to `http://a1b2c3d4e5f6g.elb.amazonaws.com` in a web browser to verify successful HTTP routing to the Nginx welcome page.

**Purpose / Why:**  
Abstracts internal cluster IPs behind a stable, internet-facing ingress endpoint, enabling load balancing across multi-replica container instances.

---

### Step 6: Automated Resource Teardown & Cleanup

**What was done:**  
Executed teardown operations to delete all active Kubernetes deployments, ELB instances, managed node groups, and the main EKS cluster to eliminate ongoing AWS charges.

**Commands executed:**
```powershell
# 1. Delete Kubernetes deployment and service (deletes associated AWS ELB)
PS C:\Users\pooja> kubectl delete -f deployment.yaml

# 2. Delete Managed Node Group and EKS Cluster
PS C:\Users\pooja> eksctl delete cluster --name devops-winter-arc-eks --region ap-south-1
```

**Purpose / Why:**  
EKS control planes, EC2 worker instances, and provisioned load balancers generate hourly usage costs. Deleting Kubernetes objects prior to cluster destruction ensures associated AWS infrastructure dependencies are fully released.

---

## Quick Reference Command Cheat Sheet

```powershell
# -------------------------------------------------------------------
# 1. CONTEXT & CLUSTER VERIFICATION
# -------------------------------------------------------------------
aws eks update-kubeconfig --region ap-south-1 --name devops-winter-arc-eks
kubectl config current-context
kubectl get nodes
kubectl get namespaces

# -------------------------------------------------------------------
# 2. MANIFEST MANAGEMENT & POD INSPECTION
# -------------------------------------------------------------------
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods -o wide
kubectl get svc nginx-demo
kubectl logs -l app=nginx-demo --all-containers

# -------------------------------------------------------------------
# 3. DISASTER RECOVERY & TROUBLESHOOTING
# -------------------------------------------------------------------
kubectl describe pod <pod-name>
kubectl describe svc nginx-demo
kubectl get events --sort-by='.metadata.creationTimestamp'

# -------------------------------------------------------------------
# 4. TEARDOWN & CLEANUP
# -------------------------------------------------------------------
kubectl delete -f deployment.yaml
eksctl delete cluster --name devops-winter-arc-eks --region ap-south-1
```

---

## Troubleshooting & Common Challenges

### Issue 1: `kubectl` Connection Failure (`Unable to connect to the server: dial tcp...`)
- **Root Cause:** Incorrect local `kubeconfig` context, region mismatch, or cluster endpoint access restricted without active AWS credentials.
- **Resolution:** Re-issue `aws eks update-kubeconfig --region ap-south-1 --name devops-winter-arc-eks` and ensure your AWS CLI identity has permissions to access the cluster API.

### Issue 2: Service `EXTERNAL-IP` Remains in `<pending>` State
- **Root Cause:** Missing VPC subnet auto-discovery tags (`kubernetes.io/role/elb = 1`) or lack of permissions for the AWS Load Balancer Controller.
- **Resolution:** Verify public subnets contain the explicit tag `kubernetes.io/role/elb = 1` and check cluster events using `kubectl get events`.

### Issue 3: Pods Stuck in `Pending` Status
- **Root Cause:** Insufficient CPU/Memory capacity on EC2 worker nodes, unresolved PersistentVolume claims, or restrictive taints/tolerations.
- **Resolution:** Execute `kubectl describe pod <pod-name>` to view scheduling event warnings, then adjust node group capacity as needed.

---

## Security Best Practices & Hardening

- **Use IRSA (IAM Roles for Service Accounts):** Avoid assigning broad IAM policies to EC2 worker nodes. Attach scoped IAM roles directly to Kubernetes service accounts.
- **Private EKS API Endpoints:** Restrict EKS cluster API endpoint access to authorized VPC CIDR blocks rather than leaving it globally public (`0.0.0.0/0`).
- **Subnet Tagging Controls:** Clearly separate public subnets (`kubernetes.io/role/elb = 1`) from internal subnets (`kubernetes.io/role/internal-elb = 1`) to prevent unintended exposure of internal workloads.
- **Automated Resource Purging:** Always delete Kubernetes `LoadBalancer` services prior to cluster termination to prevent orphaned AWS Elastic Load Balancers and unattached EBS storage volumes.

---

## Learning Outcomes & Conclusion

- **Container Orchestration at Scale:** Understood the separation between AWS-managed EKS control planes and EC2 worker node compute groups.
- **Declarative Operations:** Successfully authored and deployed production Kubernetes manifests using `Deployment` and `Service` abstractions.
- **Cloud Integrations:** Experienced native AWS integrations, linking Kubernetes `LoadBalancer` objects directly to AWS Elastic Load Balancing primitives.
- **Financial & Operational Governance:** Mastered strict lifecycle management and cleanup procedures to prevent cloud cost leaks during Kubernetes experimentation.