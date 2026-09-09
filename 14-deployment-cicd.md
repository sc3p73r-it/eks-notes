# 14. EKS Deployment and CI/CD

## Overview

A production CI/CD pipeline for EKS automates the path from source code to running pods. This includes building container images, scanning for vulnerabilities, storing in ECR, and deploying with GitOps or traditional CI/CD.

---

## Full CI/CD Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CI/CD Pipeline                                    │
│                                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌──────────┐    ┌──────────┐    │
│  │Developer │───▶│  Source      │───▶│  CI      │───▶│  Build   │    │
│  │          │    │  (GitHub)    │    │ (Actions)│    │  (Docker)│    │
│  └──────────┘    └──────────────┘    └──────────┘    └──────────┘    │
│                                                          │              │
│                                                          ▼              │
│  ┌──────────┐    ┌──────────────┐    ┌──────────┐    ┌──────────┐    │
│  │   Pods   │◀───│   Argo CD    │◀───│   GitOps │    │   ECR    │    │
│  │          │    │   (GitOps)   │    │   Repo   │    │          │    │
│  └──────────┘    └──────────────┘    └──────────┘    └──────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## GitHub Actions CI Pipeline

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app
  EKS_CLUSTER: my-cluster

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region ${{ env.AWS_REGION }}

      - name: Deploy to EKS
        run: |
          kubectl apply -f k8s/manifests/
          kubectl set image deployment/my-app my-app=${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          kubectl rollout status deployment/my-app
```

---

## Image Tagging Strategies

| Strategy | Tag | Pros | Cons |
|----------|-----|------|------|
| Immutable | `git-sha` | Reproducible, traceable | No semantic versioning |
| Semantic | `v1.2.3` | Human-readable | Requires version management |
| Date-based | `2025-01-15-1030` | Easy to sort | Not unique |
| Latest | `latest` | Simple | Non-reproducible, avoid in prod |

**Best practice**: Use the full git SHA as the image tag for immutable builds. Additionally tag with a release version like `v1.2.3` for stable releases.

---

## Helm

### Chart Structure

```
my-app-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   └── serviceaccount.yaml
└── charts/
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application
type: application
version: 1.2.3
appVersion: "1.2.3"
dependencies:
  - name: postgresql
    version: 12.1.0
    repository: https://charts.bitnami.com/bitnami
```

### Deployment Commands

```bash
# Package chart
helm package my-app-chart

# Install chart
helm install my-app ./my-app-chart --namespace production

# Upgrade chart
helm upgrade my-app ./my-app-chart --namespace production \
  --set image.tag=v1.3.0

# Rollback
helm rollback my-app 2

# List releases
helm list -A
```

---

## Kustomize

### Structure

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patch.yaml
│   └── production/
│       ├── kustomization.yaml
│       └── patch.yaml
```

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
commonLabels:
  app: my-app
```

### overlays/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - patch.yaml
images:
  - name: my-app
    newName: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app
    newTag: v1.2.3
```

```bash
# Apply with kustomize
kubectl apply -k k8s/overlays/production
```

---

## GitOps with Argo CD

### Installation

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Application Definition

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/core-services.git
    targetRevision: HEAD
    path: apps/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
```

### Deployment Strategies in Argo CD

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Rolling | Default K8s strategy | General workloads |
| Blue/Green | Parallel deploys | Zero-downtime, controlled rollout |
| Canary | Gradual traffic shift | Risk reduction |
| Progressive | Argo Rollouts with analysis | Advanced monitoring |

---

## Deployment Strategies

### Rolling Update (Default)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:v1.2.3
```

### Blue/Green

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: my-app-service
      previewService: my-app-preview
      autoPromotionEnabled: false
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v1.3.0
```

### Canary

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v1.4.0
```

---

## Rollback Strategy

```bash
# Kubernetes rollback
kubectl rollout undo deployment/my-app

# Helm rollback
helm rollback my-app 2

# GitOps rollback
# 1. Revert commit in Git repo
# 2. Argo CD auto-syncs to revert
# 3. Verify
kubectl rollout status deployment/my-app
```

---

## Production CI/CD Best Practices

| Area | Recommendation |
|------|----------------|
| Images | Use immutable tags, never `latest` in production |
| Security | Scan images before deploy, sign images |
| Secrets | Use Kubernetes Secrets or external secret manager |
| Deployment | Use blue/green or canary for critical apps |
| Rollback | Always have a rollback plan (previous image) |
| GitOps | Use Argo CD or Flux for GitOps-based deployment |
| Environments | Separate dev/staging/production clusters |
| IAM | Use short-lived credentials, OIDC authentication |
| Testing | Run integration tests before production deploy |

---

## References

- [GitHub Actions for EKS](https://docs.github.com/en/actions)
- [Helm](https://helm.sh/docs/)
- [Kustomize](https://kustomize.io/)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/)
- [EKS Blueprints](https://aws.github.io/aws-eks-best-practices/)