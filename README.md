# AWS EKS Three-Tier TODO Application

A production-style, three-tier TODO application deployed on **Amazon EKS** using **Kubernetes**, **Docker**, **Amazon ECR**, **MongoDB**, and the **AWS Load Balancer Controller**.

The application uses Python (Flask) for both the frontend and backend services, with MongoDB as the database layer.

---

## Table of Contents

- [Architecture](#architecture)
- [Project Overview](#project-overview)
- [Technologies Used](#technologies-used)
- [Application Flow](#application-flow)
- [Repository Structure](#repository-structure)
- [Deployment Guide](#deployment-guide)
- [Kubernetes Verification Commands](#kubernetes-verification-commands)
- [Troubleshooting Notes](#important-troubleshooting-learned)
- [Key Kubernetes Concepts](#important-kubernetes-concepts)
- [Cleanup](#cleanup)
- [Project Outcome](#project-outcome)
- [Contribution Guidelines](#contribution-guidelines)

---

## Architecture

```
                         Internet / User
                               |
                               v
                    AWS Application Load Balancer
                               |
                               v
                    Kubernetes Ingress (ALB)
                         /             \
                        /               \
                   /api/*                /*
                     |                    |
                     v                    v
              API Service          Frontend Service
              ClusterIP:3500       ClusterIP:3000
                     |                    |
                     v                    v
              Backend Pods           Frontend Pods
                 (2 replicas)          (2 replicas)
                     |
                     v
              MongoDB Service
              ClusterIP:27017
                     |
                     v
                MongoDB Pod
                     |
                     v
              Persistent Storage
```

<img width="1128" height="753" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/4ec3c333-fae3-40a9-8bd4-ec46e426954b" />

---

## Project Overview

This project demonstrates how to deploy a containerized three-tier application on Amazon EKS.

The three application tiers are:

| Tier | Description | Port |
|------|-------------|------|
| **Frontend** | Python Flask application serving the TODO UI | `3000` |
| **Backend / API** | Python Flask API handling business logic | `3500` |
| **Database** | MongoDB for persistent data storage | `27017` |

Kubernetes Services provide internal communication between the application components, while the **AWS Load Balancer Controller** creates an internet-facing Application Load Balancer from the Kubernetes Ingress resource.

---

## Technologies Used

- AWS EC2
- Amazon EKS
- IAM
- Amazon ECR
- AWS Application Load Balancer
- AWS Load Balancer Controller
- Kubernetes
- Docker
- Helm
- eksctl
- kubectl
- Python / Flask
- MongoDB
- PersistentVolume / PersistentVolumeClaim
- Git / GitHub

---

## Application Flow

```
User
 |
 | HTTP request
 v
AWS ALB
 |
 | Kubernetes Ingress rules
 +--------------------------+
 |                          |
 | /api/*                   | /*
 v                          v
api Service             frontend Service
 |                          |
 v                          v
Backend Pods             Frontend Pods
 |
 | MongoDB connection
 v
mongodb-svc
 |
 v
MongoDB Pod
```

- Frontend → Backend: `/api/tasks`
- Backend → MongoDB: `mongodb-svc:27017`

---

## Repository Structure

```
TWSThreeTierAppChallenge/
|
├── Application-Code-Python/
│   ├── frontend/
│   │   ├── app.py
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   ├── static/
│   │   └── templates/
│   │
│   └── backend/
│       ├── app.py
│       ├── db.py
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── models/
│       └── routes/
│
├── Kubernetes-Manifests-file/
│   ├── Frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   │
│   ├── Backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   │
│   ├── MongoDB/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ...
│   │
│   ├── ingress.yaml
│   └── ...
│
├── README.md
└── ...
```

---

## Deployment Guide

### Step 1: IAM Configuration

Create an IAM user for lab/administrative work, then configure AWS credentials:

```bash
aws configure
aws sts get-caller-identity
```

> **Note:** For production environments, avoid using `AdministratorAccess`. Use least-privilege IAM permissions and IAM roles wherever possible.

### Step 2: EC2 Setup

Launch an Ubuntu EC2 instance in the AWS region where the EKS cluster will be created, and SSH into it:

```bash
ssh -i <key.pem> ubuntu@<EC2-PUBLIC-IP>
```

This EC2 instance acts as the administration/bastion machine from which AWS CLI, kubectl, eksctl, Docker, and Helm commands are executed.

### Step 3: Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install \
  -i /usr/local/aws-cli \
  -b /usr/local/bin \
  --update

aws --version
aws configure
aws sts get-caller-identity
```

### Step 4: Install Docker

```bash
sudo apt-get update
sudo apt install docker.io -y
sudo systemctl enable --now docker

docker --version
docker ps

sudo chown $USER /var/run/docker.sock
```

### Step 5: Install kubectl

Install a `kubectl` version compatible with the EKS cluster, then verify:

```bash
kubectl version --client
kubectl get nodes   # check connectivity after cluster creation
```

### Step 6: Install eksctl

```bash
curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin/

eksctl version
```

### Step 7: Create the EKS Cluster

```bash
eksctl create cluster \
  --name three-tier-k8s \
  --region us-east-2 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2
```

Configure `kubectl`:

```bash
aws eks update-kubeconfig \
  --region us-east-2 \
  --name three-tier-k8s
```

Verify:

```bash
kubectl get nodes
kubectl get pods -A
```

Expected result:

```
NAME                           STATUS   ROLES    AGE
...                            Ready    <none>   ...
...                            Ready    <none>   ...
```

### Step 8: Deploy Kubernetes Manifests

Create the application namespace:

```bash
kubectl create namespace three-tier
```

Deploy the manifests:

```bash
kubectl apply -f Kubernetes-Manifests-file/
```

Verify:

```bash
kubectl get deploy -n three-tier
kubectl get pods -n three-tier
kubectl get svc -n three-tier
```

Expected deployments:

| Deployment | Ready |
|------------|-------|
| api        | 2/2   |
| frontend   | 2/2   |
| mongodb    | 1/1   |

Expected Services: `api`, `frontend`, `mongodb-svc`

### Step 9: Configure IAM OIDC Provider

The AWS Load Balancer Controller needs AWS permissions via IRSA. Associate an IAM OIDC provider with the EKS cluster:

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster three-tier-k8s \
  --approve
```

Verify:

```bash
aws iam list-open-id-connect-providers
```

### Step 10: Create AWS Load Balancer Controller IAM Policy

```bash
curl -O \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAM-EKS-POLICY \
  --policy-document file://iam_policy.json

AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

echo $AWS_ACCOUNT_ID
```

### Step 11: Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster three-tier-k8s \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAM-EKS-POLICY \
  --approve \
  --region us-east-2
```

Verify the ServiceAccount and IAM role:

```bash
kubectl get serviceaccount aws-load-balancer-controller -n kube-system
aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole
```

### Step 12: Install Helm

```bash
sudo snap install helm --classic
helm version

helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm repo list
```

### Step 13: Deploy AWS Load Balancer Controller

```bash
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=three-tier-k8s \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-2
```

Verify:

```bash
kubectl get deployment aws-load-balancer-controller -n kube-system
```

Expected:

```
NAME                           READY   UP-TO-DATE   AVAILABLE
aws-load-balancer-controller   2/2     2            2
```

Also check the pods:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

### Step 14: Deploy ALB Ingress

```bash
kubectl apply -f Kubernetes-Manifests-file/ingress.yaml
kubectl get ingress -n three-tier
```

Expected:

```
NAME     CLASS   HOSTS   ADDRESS                                          PORTS
mainlb   alb     *       k8s-threetie-xxxxx.us-east-2.elb.amazonaws.com   80
```

The Ingress routes:

| Path | Target |
|------|--------|
| `/api/*` | `api` Service :3500 |
| `/*` | `frontend` Service :3000 |

> A custom domain name is **not required** — the ALB DNS name can be used directly.

### Step 15: Verify the Complete Application

```bash
kubectl get deploy -n kube-system
kubectl get deploy -n three-tier
kubectl get pods -n kube-system
kubectl get pods -n three-tier
kubectl get svc -n three-tier
kubectl get ingress -n three-tier
```

Get the ALB DNS name:

```bash
kubectl get ingress mainlb \
  -n three-tier \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Test the ALB:

```bash
ALB_DNS=$(kubectl get ingress mainlb \
  -n three-tier \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo $ALB_DNS

curl -i http://$ALB_DNS/          # Frontend
curl -i http://$ALB_DNS/api/ok    # Backend health check
```

### Step 16: Test the TODO Application

Open in a browser:

```
http://<ALB-DNS>/
```

You should be able to:

- Add a TODO task
- View TODO tasks
- Update a TODO task
- Delete a TODO task

Communication paths:

- Frontend → Backend: `/api/tasks`
- Backend → MongoDB: `mongodb-svc:27017`

---

## Kubernetes Verification Commands

Useful commands during troubleshooting:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deploy -n kube-system
kubectl get deploy -n three-tier
kubectl get svc -n three-tier
kubectl get ingress -n three-tier
kubectl describe ingress mainlb -n three-tier
```

Check application logs:

```bash
kubectl logs -n three-tier deployment/api
kubectl logs -n three-tier deployment/frontend
kubectl logs -n three-tier deployment/mongodb
```

Check pod details:

```bash
kubectl describe pod <POD_NAME> -n three-tier
```

Check AWS Load Balancer Controller logs:

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

## Important Troubleshooting Learned

### 1. ALB DNS returned 404

A 404 from the ALB does not automatically mean the ALB is broken. Check the Ingress rules:

```bash
kubectl describe ingress mainlb -n three-tier
```

The request must match a configured Ingress path. The final working configuration routes:

```
/api/* -> api:3500
/*     -> frontend:3000
```

### 2. Backend `/` returned 404

The backend application did not expose `/` as the health endpoint. The correct health endpoint is `/ok`.

```bash
curl -i http://api:3500/ok
```

Expected: `HTTP/1.1 200 OK`

The backend deployment also uses `/ok` for liveness and readiness probes.

### 3. ALB target health checks returned 404

Configure the ALB health check path via annotation:

```yaml
alb.ingress.kubernetes.io/healthcheck-path: /ok
```

This lets the ALB check the backend using an endpoint that returns HTTP 200.

### 4. AWS Load Balancer Controller received `AccessDenied`

If the controller reports something like:

```
not authorized to perform: elasticloadbalancing:DescribeListenerAttributes
```

Check the IAM role and verify the correct policy is attached:

```bash
aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole
kubectl describe ingress mainlb -n three-tier
```

The Ingress should eventually show `SuccessfullyReconciled`.

### 5. `curl` was not available inside the application container

Some containers don't ship troubleshooting utilities like `curl`. Test through another pod/container that has `curl`, or test the Kubernetes Service from a dedicated debugging pod:

```bash
curl -i http://api:3500/ok
```

Expected:

```
HTTP/1.1 200 OK
...
ok
```

---

## Important Kubernetes Concepts

**Deployment** — Maintains the desired number of application Pods.

| Deployment | Replicas |
|------------|----------|
| api        | 2        |
| frontend   | 2        |
| mongodb    | 1        |

**Service** — Provides a stable internal endpoint for Pods.

| Service    | Port    |
|------------|---------|
| frontend   | `:3000` |
| api        | `:3500` |
| mongodb    | `:27017`|

**Ingress** — Defines HTTP routing rules.

```
/api/* -> api Service
/*     -> frontend Service
```

**AWS Load Balancer Controller** — Runs inside the cluster and watches Kubernetes resources such as Ingress, using AWS APIs to create and manage the Application Load Balancer and its target groups.

**OIDC + IRSA** — The OIDC provider and IAM ServiceAccount allow the AWS Load Balancer Controller Pod to assume the IAM role required to call AWS APIs:

```
Controller Pod
      |
      v
Kubernetes ServiceAccount
      |
      v
OIDC / IRSA
      |
      v
IAM Role
      |
      v
AWS ELB APIs
```

---

## Cleanup

Delete the Ingress first (allows the controller to clean up the ALB and associated resources):

```bash
kubectl delete ingress mainlb -n three-tier
```

Delete the application namespace:

```bash
kubectl delete namespace three-tier
```

Uninstall the AWS Load Balancer Controller:

```bash
helm uninstall aws-load-balancer-controller -n kube-system
```

Delete the controller ServiceAccount if required:

```bash
kubectl delete serviceaccount aws-load-balancer-controller -n kube-system
```

Detach and delete the IAM policy and role:

```bash
aws iam detach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAM-EKS-POLICY

aws iam delete-role \
  --role-name AmazonEKSLoadBalancerControllerRole

aws iam delete-policy \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAM-EKS-POLICY
```

Finally, delete the EKS cluster:

```bash
eksctl delete cluster \
  --name three-tier-k8s \
  --region us-east-2
```

Verify:

```bash
aws eks list-clusters --region us-east-2
```

> **Important:** Also check for leftover AWS resources that can keep generating charges even after the cluster is deleted — EC2 instances, EBS volumes, Load Balancers, Target Groups, NAT Gateways, Elastic IPs, Security Groups, and ECR repositories.

---

## Project Outcome

The final application runs successfully on Amazon EKS with the following stack:

```
Amazon EKS
    |
    +-- AWS Load Balancer Controller
    |
    +-- Kubernetes Ingress
    |
    +-- Frontend Deployment
    |      +-- Frontend Pod
    |      +-- Frontend Pod
    |
    +-- Backend Deployment
    |      +-- API Pod
    |      +-- API Pod
    |
    +-- MongoDB Deployment
           +-- MongoDB Pod
           +-- Persistent Storage
```

The application is accessible through the AWS ALB DNS name and supports creating, viewing, updating, and deleting TODO tasks.

---

## Contribution Guidelines

1. Fork the repository and create your feature branch.
2. Deploy the application and add your own improvements.
3. Test Kubernetes manifests before submitting changes.
4. Ensure the code follows the project's style and structure.
5. Submit a Pull Request with a clear description of your changes.
