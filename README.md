AWS EKS Three-Tier TODO Application

A three-tier TODO application deployed on Amazon EKS using Kubernetes, Docker, Amazon ECR, MongoDB, and the AWS Load Balancer Controller.

The application uses Python-based frontend and backend services and MongoDB as the database.

Architecture

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


              <img width="1128" height="753" alt="image" src="https://github.com/user-attachments/assets/4ec3c333-fae3-40a9-8bd4-ec46e426954b" />


Project Overview--
This project demonstrates how to deploy a containerized three-tier application on Amazon EKS.

The three application tiers are:
Frontend - Python Flask application serving the TODO UI.
Backend/API - Python Flask API running on port 3500.
Database - MongoDB running on port 27017.

Kubernetes Services provide internal communication between the application components, while the AWS Load Balancer Controller creates an internet-facing Application Load Balancer from the Kubernetes Ingress resource.

Technologies Used
AWS EC2
Amazon EKS
IAM
Amazon ECR
AWS Application Load Balancer
AWS Load Balancer Controller
Kubernetes
Docker
Helm
eksctl
kubectl
Python / Flask
MongoDB
PersistentVolume / PersistentVolumeClaim
Git / GitHub

Application Flow
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

The frontend communicates with the backend using:

/api/tasks

The backend communicates with MongoDB using the Kubernetes Service:
mongodb-svc:27017


Repository Structure--
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

Deployment Guide
Step 1: IAM Configuration
Create an IAM user for lab/administrative work.
Configure AWS credentials:
aws configure
Verify:
aws sts get-caller-identity

For production environments, avoid using AdministratorAccess. Use least-privilege IAM permissions and IAM roles wherever possible.

Step 2: EC2 Setup
Launch an Ubuntu EC2 instance in the AWS region where the EKS cluster will be created.
SSH into the instance:
ssh -i <key.pem> ubuntu@<EC2-PUBLIC-IP>
The EC2 instance is used as the administration/bastion machine from which AWS CLI, kubectl, eksctl, Docker and Helm commands are executed.

Step 3: Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install \
  -i /usr/local/aws-cli \
  -b /usr/local/bin \
  --update

aws --version
aws configure

Verify AWS access:

aws sts get-caller-identity

Step 4: Install Docker

sudo apt-get update

sudo apt install docker.io -y

sudo systemctl enable --now docker

docker --version
docker ps

Allow the current user to access Docker:

sudo chown $USER /var/run/docker.sock

Step 5: Install kubectl

Install a kubectl version compatible with the EKS cluster.

Verify:

kubectl version --client

Check connectivity later with:

kubectl get nodes

Step 6: Install eksctl

curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin/

eksctl version

Step 7: Create the EKS Cluster

Create the EKS cluster:

eksctl create cluster \
  --name three-tier-k8s \
  --region us-east-2 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2

Configure kubectl:

aws eks update-kubeconfig \
  --region us-east-2 \
  --name three-tier-k8s

Verify:

kubectl get nodes
kubectl get pods -A

Expected result:

NAME                           STATUS   ROLES    AGE
...                            Ready    <none>   ...
...                            Ready    <none>   ...

Step 8: Deploy Kubernetes Manifests

Create the application namespace:

kubectl create namespace three-tier

Deploy the application manifests:

kubectl apply -f Kubernetes-Manifests-file/

Verify:

kubectl get deploy -n three-tier
kubectl get pods -n three-tier
kubectl get svc -n three-tier

Expected application deployments:

NAME       READY
api        2/2
frontend   2/2
mongodb    1/1

Expected Services:

api
frontend
mongodb-svc

Step 9: Configure IAM OIDC Provider

The AWS Load Balancer Controller needs AWS permissions.

Associate an IAM OIDC provider with the EKS cluster:

eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster three-tier-k8s \
  --approve

Verify the OIDC provider:

aws iam list-open-id-connect-providers

Step 10: Create AWS Load Balancer Controller IAM Policy

Download the AWS Load Balancer Controller IAM policy:

curl -O \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

Create the IAM policy:

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAM-EKS-POLICY \
  --policy-document file://iam_policy.json

Get the AWS Account ID:

AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

echo $AWS_ACCOUNT_ID

Step 11: Create IAM Service Account

Create the IAM role and Kubernetes ServiceAccount:

eksctl create iamserviceaccount \
  --cluster three-tier-k8s \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAM-EKS-POLICY \
  --approve \
  --region us-east-2

Verify the ServiceAccount:

kubectl get serviceaccount \
  aws-load-balancer-controller \
  -n kube-system

Verify the IAM role:

aws iam get-role \
  --role-name AmazonEKSLoadBalancerControllerRole

Step 12: Install Helm

sudo snap install helm --classic

helm version

Add the EKS Helm repository:

helm repo add eks https://aws.github.io/eks-charts

helm repo update

Verify:

helm repo list

Step 13: Deploy AWS Load Balancer Controller

Install the controller:

helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=three-tier-k8s \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-2

Verify the controller:

kubectl get deployment \
  aws-load-balancer-controller \
  -n kube-system

Expected:

NAME                           READY   UP-TO-DATE   AVAILABLE
aws-load-balancer-controller   2/2     2            2

Also check the pods:

kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller

Step 14: Deploy ALB Ingress

Apply the Ingress:

kubectl apply \
  -f Kubernetes-Manifests-file/ingress.yaml

Check:

kubectl get ingress -n three-tier

Expected:

NAME     CLASS   HOSTS   ADDRESS                                      PORTS
mainlb   alb     *       k8s-threetie-xxxxx.us-east-2.elb.amazonaws.com  80

The Ingress uses an internet-facing ALB and routes:

/api/*  -> api Service :3500

/*      -> frontend Service :3000

The successful project configuration does not require a custom domain name. The ALB DNS name can be used directly.

Step 15: Verify the Complete Application

Check the Kubernetes deployments:

kubectl get deploy -n kube-system
kubectl get deploy -n three-tier

Check pods:

kubectl get pods -n kube-system
kubectl get pods -n three-tier

Check services:

kubectl get svc -n three-tier

Check Ingress:

kubectl get ingress -n three-tier

Get only the ALB DNS name:

kubectl get ingress mainlb \
  -n three-tier \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

Test the ALB:

ALB_DNS=$(kubectl get ingress mainlb \
  -n three-tier \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo $ALB_DNS

Frontend:

curl -i http://$ALB_DNS/

Backend health endpoint:

curl -i http://$ALB_DNS/api/ok

The backend application exposes /ok as its health endpoint.

Step 16: Test the TODO Application

Open the following in a browser:

http://<ALB-DNS>/

The frontend should load.

You should then be able to:

Add a TODO task

View TODO tasks

Update a TODO task

Delete a TODO task

The frontend communicates with the backend through:

/api/tasks

and the backend communicates with MongoDB through:

mongodb-svc:27017

Kubernetes Verification Commands

Useful commands during troubleshooting:

kubectl get nodes

kubectl get pods -A

kubectl get deploy -n kube-system

kubectl get deploy -n three-tier

kubectl get svc -n three-tier

kubectl get ingress -n three-tier

kubectl describe ingress mainlb -n three-tier

Check application logs:

kubectl logs -n three-tier deployment/api

kubectl logs -n three-tier deployment/frontend

kubectl logs -n three-tier deployment/mongodb

Check pod details:

kubectl describe pod <POD_NAME> -n three-tier

Check AWS Load Balancer Controller logs:

kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller

Important Troubleshooting Learned

1. ALB DNS returned 404

A 404 from the ALB does not automatically mean the ALB is broken.

Check the Ingress rules:

kubectl describe ingress mainlb -n three-tier

The request must match the configured Ingress path.

The final working configuration routes:

/api/* -> api:3500
/*     -> frontend:3000

2. Backend / returned 404

The backend application did not expose / as the health endpoint.

The correct backend health endpoint is:

/ok

Test internally:

curl -i http://api:3500/ok

Expected:

HTTP/1.1 200 OK

The Kubernetes backend deployment also uses /ok for liveness and readiness checks.

3. ALB target health checks returned 404

Configure the ALB health check path:

alb.ingress.kubernetes.io/healthcheck-path: /ok

This allows the ALB to check the backend using an endpoint that returns HTTP 200.

4. AWS Load Balancer Controller received AccessDenied

If the controller reports:

not authorized to perform:
elasticloadbalancing:DescribeListenerAttributes

check:

aws iam get-role \
  --role-name AmazonEKSLoadBalancerControllerRole

and verify that the required AWS Load Balancer Controller policy is attached.

Also check:

kubectl describe ingress mainlb -n three-tier

The Ingress should eventually show:

SuccessfullyReconciled

5. curl was not available inside the application container

A container may not contain troubleshooting utilities such as curl.

Instead, test through another pod/container that has curl, or test the Kubernetes Service from a suitable debugging pod.

Example:

curl -i http://api:3500/ok

Expected:

HTTP/1.1 200 OK
...
ok

Important Kubernetes Concepts

Deployment

Maintains the desired number of application Pods.

Example:

api        -> 2 replicas
frontend   -> 2 replicas
mongodb    -> 1 replica

Service

Provides a stable internal endpoint for Pods.

frontend  -> :3000
api       -> :3500
mongodb   -> :27017

Ingress

Defines HTTP routing rules.

/api/* -> api Service
/*     -> frontend Service

AWS Load Balancer Controller

Runs inside the Kubernetes cluster and watches Kubernetes resources such as Ingress.

It uses AWS APIs to create and manage the Application Load Balancer and its target groups.

OIDC + IRSA

The OIDC provider and IAM ServiceAccount allow the AWS Load Balancer Controller Pod to assume the IAM role required to call AWS APIs.

Flow:

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

Cleanup

Delete the Ingress first:

kubectl delete ingress mainlb -n three-tier

This gives the AWS Load Balancer Controller an opportunity to delete the ALB and associated resources.

Delete the application namespace:

kubectl delete namespace three-tier

Uninstall the AWS Load Balancer Controller:

helm uninstall aws-load-balancer-controller \
  -n kube-system

Delete the controller ServiceAccount if required:

kubectl delete serviceaccount \
  aws-load-balancer-controller \
  -n kube-system

Detach the IAM policy:

aws iam detach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAM-EKS-POLICY

Delete the IAM role:

aws iam delete-role \
  --role-name AmazonEKSLoadBalancerControllerRole

Delete the IAM policy:

aws iam delete-policy \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAM-EKS-POLICY

Finally delete the EKS cluster:

eksctl delete cluster \
  --name three-tier-k8s \
  --region us-east-2

Verify:

aws eks list-clusters \
  --region us-east-2

Also check for remaining AWS resources such as:

EC2 instances

EBS volumes

Load Balancers

Target Groups

NAT Gateways

Elastic IPs

Security Groups

ECR repositories

Important: AWS resources such as NAT Gateways, Load Balancers and EBS volumes can continue generating charges if they remain after the EKS cluster is deleted.

Project Outcome

The final application successfully runs on Amazon EKS with:

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

The application is accessible through the AWS ALB DNS name and supports creating, viewing, updating and deleting TODO tasks.

Contribution Guidelines

Fork the repository and create your feature branch.

Deploy the application and add your own improvements.

Test Kubernetes manifests before submitting changes.

Ensure the code follows the project's style and structure.

Submit a Pull Request with a clear description of your changes.
