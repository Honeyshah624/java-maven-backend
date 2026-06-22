# Java Maven Backend Deployment on AWS EKS using GitOps

Deploying a Java Maven backend application on AWS EKS using GitHub Actions for CI/CD and Argo CD for GitOps-based Kubernetes deployment.

---

## Project Overview

In this task, a Java Maven backend application is containerized using Docker, pushed to Amazon ECR, and deployed on AWS EKS using Kubernetes manifests managed through a separate GitOps repository. GitHub Actions is used to automate the CI/CD pipeline, while Argo CD continuously syncs the GitOps repository with the EKS cluster.

---

## Architecture Flow

```text
Developer pushes code to GitHub
        ↓
GitHub Actions pipeline starts
        ↓
Maven package is built
        ↓
Docker image is created
        ↓
Image is pushed to Amazon ECR
        ↓
GitHub Actions updates image tag in GitOps repo
        ↓
Argo CD detects GitOps repo change
        ↓
Argo CD syncs Kubernetes manifests to EKS
        ↓
EKS pulls latest image from ECR
        ↓
Java backend app runs inside Kubernetes pod
        ↓
Application is exposed using AWS LoadBalancer
```

---

## Technologies Used

* Java 21
* Maven
* Spring Boot
* Docker
* GitHub Actions
* Amazon ECR
* AWS EKS
* Kubernetes
* Argo CD
* Kustomize
* GitOps

---

## Repository Structure

Two separate repositories are used in this task.

```text
eks-gitops-task/
├── java-maven-backend
└── java-maven-backend-gitops
```

---

## Repository 1: Application Repo

Repository name:

```text
java-maven-backend
```

Purpose:

```text
Contains Java Maven source code, Dockerfile, and GitHub Actions workflow.
```

Structure:

```text
java-maven-backend/
├── .github/
│   └── workflows/
│       └── ci-cd-gitops.yml
├── .mvn/
│   └── wrapper/
│       └── maven-wrapper.properties
├── src/
│   ├── main/
│   ├── test/
│   └── checkstyle/
├── .dockerignore
├── .editorconfig
├── .gitattributes
├── .gitignore
├── Dockerfile
├── mvnw
├── mvnw.cmd
├── pom.xml
└── README.md
```

---

## Repository 2: GitOps Repo

Repository name:

```text
java-maven-backend-gitops
```

Purpose:

```text
Contains Kubernetes manifests and Argo CD Application configuration.
```

Structure:

```text
java-maven-backend-gitops/
├── apps/
│   └── backend/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           └── dev/
│               └── kustomization.yaml
└── argocd/
    └── backend-application.yaml
```

---

## Prerequisites

Before starting this task, the following tools and accounts are required:

```text
AWS account
GitHub account
AWS CLI
kubectl
eksctl
Docker
Git
Argo CD CLI
Java 21
Maven or Maven Wrapper
```

Check installed tools:

```bash
aws --version
kubectl version --client
eksctl version
docker --version
git --version
java --version
```

---

## Step 1: Configure AWS CLI

Configure AWS CLI with personal AWS account credentials:

```bash
aws configure
```

Verify active AWS account:

```bash
aws sts get-caller-identity
```

Set environment variables:

```bash
export AWS_REGION=ap-south-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REPOSITORY=java-maven-backend
```

---

## Step 2: Dockerize the Java Maven App

Docker image was built using a Dockerfile.

Build image locally:

```bash
docker build -t java-maven-backend:local .
```

Test Docker container locally:

```bash
docker run -p 9090:8080 java-maven-backend:local
```

Open:

```text
http://localhost:9090
```

---

## Step 3: Create Amazon ECR Repository

Create ECR repository:

```bash
aws ecr create-repository \
  --repository-name java-maven-backend \
  --region ap-south-1 \
  --image-scanning-configuration scanOnPush=true
```

Verify repository:

```bash
aws ecr describe-repositories \
  --repository-names java-maven-backend \
  --region ap-south-1
```

---

## Step 4: Push Docker Image to Amazon ECR

Login to ECR:

```bash
aws ecr get-login-password --region ap-south-1 | docker login \
  --username AWS \
  --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com
```

Tag Docker image:

```bash
docker tag java-maven-backend:local \
  ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/java-maven-backend:v1
```

Push Docker image:

```bash
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/java-maven-backend:v1
```

Verify image:

```bash
aws ecr list-images \
  --repository-name java-maven-backend \
  --region ap-south-1
```

---

## Step 5: Create EKS Cluster

Create EKS cluster:

```bash
eksctl create cluster \
  --name java-gitops-eks \
  --region ap-south-1 \
  --nodegroup-name app-ng \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed
```

Update kubeconfig:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name java-gitops-eks
```

Verify nodes:

```bash
kubectl get nodes
```

Expected output:

```text
STATUS
Ready
```

---

## Step 6: Install Argo CD

Create namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check Argo CD pods:

```bash
kubectl get pods -n argocd
```

Expected:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-redis
argocd-dex-server
```

---

## Step 7: Access Argo CD UI

Port-forward Argo CD server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8082:443
```

Open:

```text
https://localhost:8082
```

Get Argo CD initial password:

```bash
argocd admin initial-password -n argocd
```

Login:

```text
Username: admin
Password: <initial-password>
```

CLI login:

```bash
argocd login localhost:8082 \
  --username admin \
  --password <initial-password> \
  --insecure
```

---

## Step 8: Create Argo CD Application

Apply Argo CD Application YAML:

```bash
kubectl apply -f argocd/backend-application.yaml
```

Check application:

```bash
kubectl get applications -n argocd
```

Sync application:

```bash
argocd app sync java-maven-backend-dev
```

Check status:

```bash
argocd app get java-maven-backend-dev
```

Expected:

```text
Sync Status: Synced
Health Status: Healthy
```

---

## Step 9: Verify Kubernetes Deployment

Check backend namespace resources:

```bash
kubectl get all -n backend-dev
```

Check pod status:

```bash
kubectl get pods -n backend-dev
```

Expected:

```text
java-maven-backend-xxxxx   1/1   Running
```

Check service:

```bash
kubectl get svc java-maven-backend -n backend-dev
```

Access application:

```text
http://<LOADBALANCER-DNS>
```

---

## Step 10: GitHub Actions CI/CD Pipeline

GitHub Actions workflow file:

```text
.github/workflows/ci-cd-gitops.yml
```

Pipeline steps:

```text
Checkout app repository
Make Maven wrapper executable
Set up Java 21
Build Maven package
Configure AWS credentials
Login to Amazon ECR
Build Docker image
Push Docker image to ECR
Checkout GitOps repository
Update image tag in GitOps repo
Push GitOps repo changes
```

---

## GitHub Actions Secrets

Added in application repo:

```text
Settings → Secrets and variables → Actions → Secrets
```

Secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
GITOPS_TOKEN
```

---

## GitHub Actions Variables

Added in application repo:

```text
Settings → Secrets and variables → Actions → Variables
```

Variables:

```text
AWS_REGION = ap-south-1
AWS_ACCOUNT_ID = <AWS_ACCOUNT_ID>
ECR_REPOSITORY = java-maven-backend
GITOPS_REPO = Honeyshah624/java-maven-backend-gitops
```

---

## GitHub Actions Workflow

```yaml
name: Java Maven CI CD GitOps

on:
  push:
    branches:
      - main

permissions:
  contents: read

env:
  AWS_REGION: ${{ vars.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ vars.AWS_ACCOUNT_ID }}
  ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
  GITOPS_REPO: ${{ vars.GITOPS_REPO }}

jobs:
  build-push-update-gitops:
    name: Build Push Update GitOps
    runs-on: ubuntu-latest

    steps:
      - name: Checkout app repo
        uses: actions/checkout@v4

      - name: Make Maven wrapper executable
        run: chmod +x ./mvnw

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Build Maven package
        run: ./mvnw -B clean package -DskipTests

      - name: Configure AWS credentials using access keys
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set image tag
        run: |
          echo "IMAGE_TAG=${GITHUB_SHA::7}" >> $GITHUB_ENV
          echo "ECR_REGISTRY=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com" >> $GITHUB_ENV

      - name: Build Docker image
        run: |
          docker build \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            .

      - name: Push Docker image to ECR
        run: |
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Update image tag in GitOps repo
        run: |
          cd gitops/apps/backend/overlays/dev

          sed -i "s|newTag:.*|newTag: ${IMAGE_TAG}|g" kustomization.yaml

          cd ../../../..

          git config user.name "github-actions"
          git config user.email "github-actions@github.com"

          git add apps/backend/overlays/dev/kustomization.yaml
          git commit -m "Update backend image tag to ${IMAGE_TAG}" || echo "No changes to commit"
          git push
```

---

## Step 11: Final CI/CD Flow

Once code is pushed to the `main` branch:

```text
GitHub Actions starts
↓
Maven package is built
↓
Docker image is created
↓
Image is pushed to ECR with commit SHA tag
↓
GitOps repo kustomization.yaml is updated
↓
Argo CD detects the GitOps change
↓
Argo CD deploys latest image to EKS
```

---

## Troubleshooting

### Issue 1: EKS Nodegroup Failed

Reason:

```text
Instance type was not eligible for the account/free-tier usage.
```

Fix:

```text
Used t3.micro or t3.small based on available eligible instance types.
```

Command used to check eligible instance types:

```bash
aws ec2 describe-instance-types \
  --region ap-south-1 \
  --filters "Name=free-tier-eligible,Values=true" \
  --query "InstanceTypes[*].InstanceType" \
  --output table
```

---

### Issue 2: Argo CD Pods Pending

Reason:

```text
Micro nodes had too many pods or low capacity.
```

Fix:

```text
Created t3.small nodegroup.
```

---

### Issue 3: Argo CD UI Login Failed

Reason:

```text
Wrong password or old Argo CD session.
```

Fix:

```bash
argocd admin initial-password -n argocd
```

---

### Issue 4: ImagePullBackOff

Reason:

```text
Kubernetes was trying to pull wrong image: java-maven-backendcd
```

Fix:

```text
Corrected image name to java-maven-backend in deployment.yaml.
```

Correct image:

```text
<AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/java-maven-backend:v1
```

---

### Issue 5: LoadBalancer Timeout

Reason:

```text
Pod was not running because of ImagePullBackOff, so service endpoints were empty.
```

Fix:

```text
Fixed image path and synced Argo CD again.
```

Check endpoints:

```bash
kubectl get endpoints java-maven-backend -n backend-dev
```

---

## Useful Commands

Check EKS nodes:

```bash
kubectl get nodes
```

Check Argo CD pods:

```bash
kubectl get pods -n argocd
```

Check Argo CD app:

```bash
argocd app get java-maven-backend-dev
```

Sync Argo CD app:

```bash
argocd app sync java-maven-backend-dev
```

Check backend pods:

```bash
kubectl get pods -n backend-dev
```

Check backend service:

```bash
kubectl get svc -n backend-dev
```

Check deployed image:

```bash
kubectl get deployment java-maven-backend -n backend-dev -o yaml | grep image:
```

Check ECR images:

```bash
aws ecr list-images \
  --repository-name java-maven-backend \
  --region ap-south-1
```

---

## Cleanup

Since this task was performed in a personal AWS account, cleanup is recommended after demo.

Delete Argo CD application:

```bash
kubectl delete application java-maven-backend-dev -n argocd
```

Delete namespaces:

```bash
kubectl delete namespace backend-dev
kubectl delete namespace argocd
```

Delete EKS cluster:

```bash
eksctl delete cluster \
  --name java-gitops-eks \
  --region ap-south-1
```

Delete ECR repository:

```bash
aws ecr delete-repository \
  --repository-name java-maven-backend \
  --region ap-south-1 \
  --force
```

---

## Final Outcome

The Java Maven backend application was successfully deployed on AWS EKS using a complete GitOps CI/CD workflow. GitHub Actions handled the build and image push process, Amazon ECR stored the Docker images, the GitOps repository maintained Kubernetes desired state, and Argo CD automatically deployed the latest application version to EKS.
