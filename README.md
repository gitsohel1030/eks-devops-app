# eks-devops-app

Application source repository for the EKS DevOps Platform. Contains the Dockerfile, Jenkins pipeline, and application code for the FoodRush web application. This repo triggers the CI/CD pipeline — every push to `main` results in a new Docker image being built, pushed to ECR, and deployed to EKS via GitOps.

---

## Table of Contents

- [Repository Structure](#repository-structure)
- [Application](#application)
- [CI/CD Pipeline](#cicd-pipeline)
- [Pipeline Stages](#pipeline-stages)
- [Blue/Green Deployment Strategy](#bluegreen-deployment-strategy)
- [Pipeline Parameters](#pipeline-parameters)
- [Environment Variables](#environment-variables)
- [Jenkins Setup Requirements](#jenkins-setup-requirements)
- [Workflow](#workflow)

---

## Repository Structure

```
eks-devops-app/
│
├── Dockerfile          # Container image definition
├── Jenkinsfile         # CI/CD pipeline definition
└── app/
    ├── index.html      # Main application page
    ├── health.html     # Health check endpoint
    └── foodrush.zip    # Application assets
```

---

## Application

A containerised static web application served via nginx, used as the deployment vehicle for demonstrating production-grade DevOps practices on EKS.

| Property | Value |
|---|---|
| Base Image | `nginxinc/nginx-unprivileged:alpine` |
| Port | `8080` |
| Health Check | `GET /health.html` → `200 OK` |
| Security | Non-root user, read-only filesystem |

### Dockerfile

```dockerfile
FROM nginxinc/nginx-unprivileged:alpine
COPY app/ /usr/share/nginx/html
EXPOSE 8080
```

**Why `nginx-unprivileged`?**
Standard nginx binds to port 80 which requires root privileges. The unprivileged variant runs as a non-root user on port 8080 — required for Kubernetes pods with `runAsNonRoot: true` security context.

---

## CI/CD Pipeline

The Jenkins pipeline implements a **GitOps-based blue/green deployment** — it never applies changes directly to Kubernetes. Instead it updates the gitops repository, and ArgoCD reconciles the cluster state.

### High Level Flow

```
Git Push to main
      ↓
Jenkins triggered (webhook)
      ↓
Build Docker image tagged with Git commit SHA
      ↓
Push image to Amazon ECR
      ↓
Clone eks-devops-gitops repo
      ↓
Read release.yaml → determine active color
      ↓
Update TARGET color image patch
      ↓
Update traffic routing in kustomization.yaml
      ↓
Push changes to eks-devops-gitops repo
      ↓
ArgoCD detects change → deploys new color
      ↓
ALB routes 100% traffic to new color
```

### Pipeline Architecture Diagram

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
│   app-repo   │     │    Jenkins   │     │      AWS             │
│              │     │              │     │                      │
│  code push ──┼────▶│  build image │────▶│  ECR (image stored)  │
│              │     │              │     │                      │
└──────────────┘     └──────┬───────┘     └──────────────────────┘
                            │
                            │ update image tag
                            ▼
                  ┌──────────────────┐
                  │  gitops-repo     │
                  │                  │
                  │  release.yaml    │
                  │  patch-*.yaml    │
                  │  kustomization   │
                  └────────┬─────────┘
                           │
                           │ ArgoCD detects change
                           ▼
                  ┌──────────────────┐
                  │   EKS Cluster    │
                  │                  │
                  │  web-app-blue    │
                  │  web-app-green ◀─┼── new color deployed
                  │                  │
                  │  ALB → 100%      │
                  │  traffic to new  │
                  └──────────────────┘
```

---

## Pipeline Stages

### Stage 1 — Checkout
Checks out the application source code from this repository.

---

### Stage 2 — Set IMAGE_TAG from commit
```
IMAGE_TAG = first 7 characters of GIT_COMMIT SHA
Example: a3f9b2c
```
Using the Git commit SHA as the image tag ensures:
- Every image is uniquely and immutably tagged
- You can trace any running pod back to its exact source commit
- No ambiguity with `latest` tags

---

### Stage 3 — Build & Push Image to ECR
```
IF image with this tag already exists in ECR:
    Skip build (idempotent — safe to re-run)
ELSE:
    docker build -t {ECR_REPO}:{IMAGE_TAG} .
    aws ecr get-login-password | docker login
    docker push {ECR_REPO}:{IMAGE_TAG}
```

The existence check makes the pipeline **idempotent** — re-running the same commit never rebuilds an already-pushed image.

---

### Stage 4 — Configure Kubectl
```
aws eks update-kubeconfig --region ap-south-1 --name my-eks-cluster-1030
```
Configures kubectl context to point at the EKS cluster. Jenkins uses an IAM role (`jenkins-eks-role`) with cluster admin access.

---

### Stage 5 — Ensure Namespace
```
kubectl get ns prod-app || kubectl create ns prod-app
```
Idempotent namespace creation — safe to run on every build.

---

### Stage 6 — Clone GitOps Repo
Clones `eks-devops-gitops` using SSH key credential (`github-ssh-gitops`). This gives Jenkins read/write access to the gitops repository without exposing credentials.

---

### Stage 7 — Determine TARGET Color
```
Read eks-devops-gitops/k8s/overlays/prod/release.yaml
activeColor: green → TARGET = blue
activeColor: blue  → TARGET = green
```

| Current Active | Deploy Target |
|---|---|
| `blue` | `green` |
| `green` | `blue` |

---

### Stage 8 — Push Changes in Git
Updates three files in the gitops repo:

| File | Change |
|---|---|
| `k8s/overlays/prod/release.yaml` | Sets `activeColor` to TARGET color |
| `k8s/overlays/prod/patch-{TARGET}-image.yaml` | Updates image tag to new ECR image |
| `k8s/overlays/prod/kustomization.yaml` | Switches traffic patch to TARGET color |

Commit message format:
```
GitOps deploy {IMAGE_TAG} to {TARGET_COLOR}
```

---

## Blue/Green Deployment Strategy

### How Traffic Switches

The ALB routes traffic using weighted target groups. The `traffic/` patches in the gitops repo define the weights:

```
Initial state:   blue=100%, green=0%
After deploy:    blue=0%,   green=100%
After rollback:  blue=100%, green=0%
```

Traffic switching is **instantaneous** — the ALB updates weights atomically with no downtime.

### Deployment States

```
State 1: Blue Active
┌─────────────┐     ┌─────────────┐
│  ALB        │────▶│  Blue Pods  │ ← 100% traffic
│             │     └─────────────┘
│             │     ┌─────────────┐
│             │     │ Green Pods  │ ← 0% traffic (scaled down)
└─────────────┘     └─────────────┘

State 2: Deploying to Green
┌─────────────┐     ┌─────────────┐
│  ALB        │────▶│  Blue Pods  │ ← 100% traffic (still active)
│             │     └─────────────┘
│             │     ┌─────────────┐
│             │     │ Green Pods  │ ← 0% traffic (new image deploying)
└─────────────┘     └─────────────┘

State 3: Green Active
┌─────────────┐     ┌─────────────┐
│  ALB        │     │  Blue Pods  │ ← 0% traffic
│             │     └─────────────┘
│             │     ┌─────────────┐
│             │────▶│ Green Pods  │ ← 100% traffic (new version)
└─────────────┘     └─────────────┘
```

---

## Pipeline Parameters

The pipeline supports three operational modes via build parameters:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `PROMOTE` | Boolean | false | Promote TARGET color image to both colors (baseline sync) |
| `ROLLBACK` | Boolean | false | Revert last commit in gitops repo |

### Normal Deployment
```
PROMOTE = false
ROLLBACK = false
```
Standard build → push → deploy flow.

---

### Promote (Manual)
```
PROMOTE = true
ROLLBACK = false
```

When you're confident the new deployment is stable, promote applies the TARGET color's image tag to the other color as well — ensuring both are in sync for the next deployment cycle.

```
Before promote:
  blue  → image: web-app:abc1234  (old)
  green → image: web-app:def5678  (new, active)

After promote:
  blue  → image: web-app:def5678  (synced)
  green → image: web-app:def5678  (active)
```

---

### Rollback (Manual)
```
PROMOTE = false
ROLLBACK = true
```

Performs a `git revert HEAD` on the gitops repo — undoing the last deployment commit. ArgoCD detects the revert and restores the previous state automatically.

```
git revert HEAD
      ↓
gitops repo reverts to previous image tags + traffic patch
      ↓
ArgoCD syncs cluster back to previous state
      ↓
ALB traffic returns to previous color
```

---

## Environment Variables

| Variable | Value | Description |
|---|---|---|
| `AWS_REGION` | `ap-south-1` | AWS region |
| `CLUSTER_NAME` | `my-eks-cluster-1030` | EKS cluster name |
| `ACCOUNT_ID` | `608827180555` | AWS account ID |
| `REPO_NAME` | `web-app` | ECR repository name |
| `ECR_REPO` | `{ACCOUNT_ID}.dkr.ecr.{REGION}.amazonaws.com/web-app` | Full ECR URI |
| `IMAGE_TAG` | `{GIT_COMMIT[0:7]}` | Git commit SHA (7 chars) |
| `K8S_NAMESPACE` | `prod-app` | Kubernetes deployment namespace |
| `GITOPS_REPO` | SSH URL of gitops repo | GitOps repository |
| `GITOPS_DIR` | `eks-devops-gitops` | Local clone directory |

---

## Jenkins Setup Requirements

### IAM Role

Jenkins EC2 instance must have an IAM role (`jenkins-eks-role`) with:

| Permission | Purpose |
|---|---|
| `ecr:GetAuthorizationToken` | Login to ECR |
| `ecr:BatchCheckLayerAvailability` | Check if image exists |
| `ecr:PutImage` | Push image layers |
| `ecr:InitiateLayerUpload` | Push image |
| `ecr:UploadLayerPart` | Push image |
| `ecr:CompleteLayerUpload` | Push image |
| `ecr:DescribeImages` | Check if tag exists |
| `eks:DescribeCluster` | Update kubeconfig |

The Jenkins IAM role must also be mapped as cluster admin in EKS Access Entries.

### Jenkins Credentials

| Credential ID | Type | Purpose |
|---|---|---|
| `github-ssh-gitops` | SSH Private Key | Read/write access to gitops repo |

### Required Jenkins Plugins

| Plugin | Purpose |
|---|---|
| Pipeline | Jenkinsfile support |
| Git | Source code checkout |
| AWS Credentials | AWS integration |
| SSH Agent | Git SSH operations |
| Kubernetes CLI | kubectl operations |

---

## Workflow

### First Time Setup

```bash
# 1. Configure Jenkins job pointing to this repo
# 2. Add github-ssh-gitops SSH credential in Jenkins
# 3. Ensure Jenkins EC2 has correct IAM role
# 4. Run pipeline with PROMOTE=false, ROLLBACK=false
```

### Day to Day Development

```bash
# Make changes to application
vim app/index.html

# Commit and push — pipeline triggers automatically
git add .
git commit -m "feat: update homepage content"
git push origin main

# Monitor in Jenkins console output
# Check ArgoCD UI for deployment status
# Verify in Grafana that new pods are healthy
```

### Emergency Rollback

```
Jenkins → Build with Parameters
  ROLLBACK = true
  PROMOTE  = false
→ Click Build
→ Previous version restored in ~2 minutes
```

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [aws-eks-devops-platform](https://github.com/YOUR_USERNAME/aws-eks-devops-platform) | Terraform infrastructure — EKS, VPC, IAM, ArgoCD install |
| [eks-devops-gitops](https://github.com/YOUR_USERNAME/eks-devops-gitops) | GitOps manifests — ArgoCD Applications, Kubernetes configs, Helm values |
