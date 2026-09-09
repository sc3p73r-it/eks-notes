# 15. Container Registry (Amazon ECR)

## Overview

Amazon Elastic Container Registry (ECR) is a fully managed container image registry that integrates natively with EKS. Images stored in ECR are pulled by nodes through the container runtime (containerd).

---

## ECR Concepts

| Term | Description |
|------|-------------|
| **Repository** | A collection of image manifests for a single container |
| **Image** | A snapshot of the container filesystem + metadata |
| **Image tag** | Human-readable reference to an image (e.g., `v1.2.3`) |
| **Image digest** | SHA-256 cryptographic hash of the image manifest |
| **Manifest** | JSON describing image layers and metadata |

---

## Complete Image Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Build → Registry → Run Flow                           │
│                                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │
│  │  Source  │───▶│  Docker  │───▶│   ECR    │───▶│   Node   │        │
│  │  Code    │    │  Build   │    │ Registry │    │ (kubelet)│        │
│  └──────────┘    └──────────┘    └──────────┘    └────┬─────┘        │
│                                                       │               │
│                                                       ▼               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────────────────┐  │
│  │ Continue │◀───│  Image   │◀───│ containerd│    │  Pull via       │  │
│  │  Process │    │ Manifest │    │ (runtime)│    │  VPC Endpoint    │  │
│  └──────────┘    └──────────┘    └──────────┘    └─────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ECR Repository Operations

```bash
# Create repository
aws ecr create-repository \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS \
  --image-tag-mutability IMMUTABLE

# List repositories
aws ecr describe-repositories

# List images
aws ecr describe-images --repository-name my-app

# Get image digest
aws ecr describe-images --repository-name my-app \
  --image-ids imageTag=v1.2.3

# Docker login
aws ecr get-login-password --region us-east-1 | docker login \
  --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

---

## Image Lifecycle Policy

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire images older than 30 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 30
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Keep last 10 images per tag prefix",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v", "release"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    }
  ]
}
```

```bash
# Apply lifecycle policy
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --policy-text file://lifecycle-policy.json
```

---

## Image Scanning

### Basic Scanning (ECR-native)

```bash
aws ecr start-image-scan --repository-name my-app --image-id imageTag=v1.2.3

aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=v1.2.3
```

### Enhanced Scanning (Amazon Inspector)

```bash
# Enable enhanced scanning
aws ecr put-registry-scanning-configuration \
  --scanning-configuration \
  '{"ScanType":"ENHANCED"}'
```

---

## Cross-Account Access

### Repository Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountPull",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::567890123456:root"
      },
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }
  ]
}
```

```bash
aws ecr set-repository-policy \
  --repository-name my-app \
  --policy-text file://repo-policy.json
```

---

## Cross-Region Replication

```bash
# Create replication configuration
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [{
        "region": "eu-west-1",
        "registryId": "123456789012"
      }],
      "repositoryFilters": [{
        "filterType": "PREFIX_MATCH",
        "filter": "prod"
      }]
    }]
  }'
```

---

## Pull-Through Cache

```bash
# Create pull-through cache rule for Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix docker.io \
  --upstream-registry-url registry-1.docker.io

# Create pull-through cache rule for ECR Public
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix ecr-public \
  --upstream-registry-url public.ecr.aws
```

Pull the image through the cache:

```bash
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/docker.io/library/nginx:latest
```

The image cache is then stored in ECR with digests preserved.

---

## EKS Integration

### Pull from ECR in a Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
          imagePullPolicy: IfNotPresent
```

### Node IAM Permissions for ECR

The node IAM role needs these permissions (included in `AmazonEKSWorkerNodePolicy` context or attached separately):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Encryption

ECR supports two encryption modes:

| Mode | Description | Use Case |
|------|-------------|----------|
| AES-256 (default) | AWS-managed key | General workloads |
| KMS | Customer-managed key | Compliance, auditing, key rotation control |

```bash
# Create KMS key for ECR
aws kms create-key --description "ECR Encryption Key"

# Create repository with KMS encryption
aws ecr create-repository \
  --repository-name my-app \
  --encryption-configuration \
  encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123456789012:key/abc-123
```

---

## Cost Control

| Practice | Benefit |
|----------|---------|
| Lifecycle policies | Purge untagged/old images, reduce storage cost |
| Tag immutability | Prevent accidental overwrite, reproducible builds |
| Replicate only prod repos | Reduce cross-region storage cost |
| Pull-through cache | Reduce egress for repeated upstream pulls |
| Image layer reuse | Share base layers, reduce storage |
| Registry scanning on push | Prevent vulnerable images from being stored |

---

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| Tagging | Use immutable tags (git SHA), never `latest` in prod |
| Scanning | Enable scanOnPush, use Amazon Inspector enhanced scanning |
| Encryption | Use KMS encryption for image data |
| Lifecycle | Set lifecycle policies to prune old images |
| Access | Use repository policies for cross-account, treat EC2 nodes with minimal pull perms |
| Replication | Use cross-region replication for DR regions |
| Pull-through cache | Cache upstream images to avoid rate limits and egress |

---

## Troubleshooting

```bash
# Check if node can authenticate to ECR
aws ecr get-authorization-token --region us-east-1

# Check image pull errors
kubectl describe pod <pod-name> | grep -A10 "Events"

# Verify ECR permissions
aws sts get-caller-identity
aws ecr describe-images --repository-name my-app

# Test pull manually on a node
sudo ctr images pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3

# Check ECR VPC endpoint
aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.us-east-1.ecr.dkr"
```

---

## References

- [ECR User Guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
- [ECR with EKS](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ECR_on_EKS.html)
- [ECR Lifecycle Policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/lifecycle_policy.html)