# 🚀 Ultimate DevOps Project – End-to-End Microservices Deployment

## 📌 Overview

This project demonstrates a complete **end-to-end DevOps implementation** for a microservices-based application using modern tools and best practices.

It covers:

* Application build (Go microservices)
* Containerization using Docker
* CI using GitHub Actions
* Infrastructure provisioning using Terraform (AWS)
* Kubernetes deployment (EKS)
* Ingress & Load Balancing (ALB)
* Domain setup using Route 53
* Continuous Delivery using ArgoCD

---

## 🏗️ Architecture

* Microservices (Go-based)
* Dockerized applications
* AWS EKS Cluster
* AWS ALB Ingress Controller
* Route 53 for DNS
* ArgoCD for GitOps deployment

---

## ⚙️ Prerequisites

* AWS Account
* Docker installed
* Terraform installed
* kubectl configured
* eksctl installed
* Helm installed
* GitHub repository
* Domain (e.g., `devopsbysai.website`)

---

## 📦 Step 1: Build Microservice Locally

```bash
cd src/product-catalog

export PRODUCT_CATALOG_PORT=<any-unique-port>
go build -o product-catalog .
```

---

## 🐳 Step 2: Dockerize the Application

### Dockerfile (Multi-stage Build)

```dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /usr/src/app/

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    mkdir -p /root/.cache/go-build

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o product-catalog .

####################################

FROM alpine AS release

WORKDIR /usr/src/app/

COPY ./products/ ./products/
COPY --from=builder /usr/src/app/product-catalog ./

ENV PRODUCT_CATALOG_PORT=8088
ENTRYPOINT ["./product-catalog"]
```

---

## 🏷️ Step 3: Build & Push Docker Image

```bash
docker build -t saitejadst/product-catalog:v1 .
docker push saitejadst/product-catalog:v1
```

<img width="1742" height="628" alt="image" src="https://github.com/user-attachments/assets/f80267b4-d206-4efb-9cb0-badd99ea169e" />


### Push All Images (Script)

```bash
for img in $(docker images --format "{{.Repository}}:{{.Tag}}" | grep saitejadst); do
  docker push $img
done
```

---

## ☁️ Step 4: Terraform Backend (S3 + DynamoDB)

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "saitejas3-eks-12345"

  lifecycle {
    prevent_destroy = false
  }
}

resource "aws_dynamodb_table" "my_table" {
  name         = "saitejas3-eks-12345"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```bash
terraform init
terraform plan
terraform apply
```

---

<img width="1002" height="592" alt="image" src="https://github.com/user-attachments/assets/15021ad2-b45b-4e4f-b8e5-410a2d5777c8" />
<img width="1675" height="562" alt="image" src="https://github.com/user-attachments/assets/2ff452f8-7955-4769-94a6-479cac3efe0a" />



## 🌐 Step 5: Create EKS Cluster (Terraform Modules)

After cluster creation:

```bash
aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster
kubectl get nodes
```
<img width="861" height="135" alt="image" src="https://github.com/user-attachments/assets/ab549d25-4a58-411a-aa9f-5918358d40a8" />


---

## ☸️ Step 6: Deploy Application to Kubernetes

### Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: opentelemetry-demo
```

```bash
kubectl apply -f serviceaccount.yaml
kubectl get sa
```

---

### Deploy Services

```bash
kubectl apply -f completedeploy.yaml
kubectl get pods
```
<img width="946" height="351" alt="image" src="https://github.com/user-attachments/assets/e87bc02e-fa8b-49ed-a738-d436f4fe6133" />


---

## 🌍 Step 7: Expose Application (LoadBalancer)

```bash
kubectl edit svc opentelemetry-demo-frontendproxy
```

Change:

```
type: LoadBalancer
```

Access via external URL (port 8080)

---
<img width="1791" height="936" alt="image" src="https://github.com/user-attachments/assets/ff511393-dc99-46a4-b527-1d2c0f19bc64" />


## 🔀 Step 8: Setup ALB Ingress Controller

### IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### IAM Role

```bash
eksctl create iamserviceaccount \
  --cluster=<cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### Install via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<vpc-id>
```

---

## 🌐 Step 9: Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-proxy
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - host: devopsbysai.website
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

```bash
kubectl apply -f ingress.yaml
kubectl get ing
```
<img width="1227" height="176" alt="image" src="https://github.com/user-attachments/assets/37b07ba1-8b19-4149-93dd-8b004b0088f2" />


---

## 🌍 Step 10: Domain Setup (Route 53)

* Create Hosted Zone: `devopsbysai.website`
* Update nameservers at domain provider
* Create A (Alias) → ALB

⏳ Wait 1–48 hours for DNS propagation

---

## 🔄 Step 11: CI Pipeline (GitHub Actions)

Pipeline stages:

* Build
* Test
* Code Quality
* Docker Build & Push
* Update Kubernetes Manifest

```yaml
on:
  pull_request:
    branches:
      - main
complete file can be found here https://github.com/saitejadst/ultimate-devops-project-demo/blob/main/.github/workflows/productcatlogci.yaml
```

Key features:

* Builds Go app
* Runs tests
* Pushes Docker image
* Updates deployment YAML with new image tag

  <img width="1232" height="460" alt="image" src="https://github.com/user-attachments/assets/0312ed9c-e5da-4d2f-86ef-4fe4d990cd7f" />


---

## 🚀 Step 12: CD using ArgoCD

### Install ArgoCD

```bash
kubectl create namespace argocd
```

Follow official docs:
https://argo-cd.readthedocs.io/en/stable/

---

### Access ArgoCD

```bash
kubectl get svc -n argocd
```

---

### Get Password

```bash
kubectl get secrets -n argocd
kubectl edit secret argocd-initial-admin-secret -n argocd
echo <password> | base64 --decode
```

Login:

* Username: `admin`
* Password: decoded value

---

### Deploy via ArgoCD

* Connect GitHub repo
* Select path: `kubernetes/productcatalog`
* Enable auto-sync

---

## 🔄 Step 13: GitOps Flow

1. Raise PR
2. CI pipeline updates image tag
3. Merge to main
4. ArgoCD detects change
5. Auto-sync deploys to Kubernetes

<img width="1586" height="746" alt="image" src="https://github.com/user-attachments/assets/8e738b5e-8b10-41cd-b235-98a984bb0db2" />
<img width="1142" height="821" alt="image" src="https://github.com/user-attachments/assets/ab4d9f70-59b9-41ca-b80f-6b6646d96ca1" />
<img width="1642" height="802" alt="image" src="https://github.com/user-attachments/assets/a6582d6b-1ffb-4255-834a-82ff638c4c05" />



---

## ✅ Final Outcome

* Microservices deployed on EKS
* Accessible via domain:

  ```
  http://www.devopsbysai.website
  ```
* CI/CD fully automated
* GitOps workflow implemented

---

## 🧠 Key Learnings

* Multi-stage Docker builds
* Terraform remote backend (S3 + DynamoDB)
* EKS cluster provisioning
* ALB Ingress Controller setup
* Route 53 DNS management
* GitHub Actions CI pipeline
* ArgoCD GitOps deployment

---

## 📌 Author

**Saiteja Dosapati**

---
