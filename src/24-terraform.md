# 24. Terraform for EKS

## Overview

Terraform is the standard Infrastructure as Code (IaC) tool for provisioning EKS. Best practices include modular design, remote state, separate environments, and GitOps for Kubernetes resources on top of Terraform-managed infrastructure.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Terraform EKS Architecture                          │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    Terraform State (S3 backend)                    │  │
│  │                    Locking via DynamoDB                            │  │
│  └───────────────┬───────────────────────────────────────────────────┘ │
│                  │                                                     │
│  ┌───────────────▼───────────────────────────────────────────────────┐ │
│  │                          Modules                                   │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐│ │
│  │  │   VPC    │ │  EKS     │ │  IAM     │ │  Nodes   │ │ Add-ons ││ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘│ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐│ │
│  │  │ Security │ │   KMS    │ │  ECR     │ │   ALB    │ │ Route53 ││ │
│  │  │  Groups  │ │          │ │          │ │          │ │ + ACM   ││ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘│ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  Terraform State → Plan → Apply → EKS infrastructure                   │
│                                                                         │
│  Kubernetes resources (Deployments) → managed by GitOps (Argo CD)      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
eks-platform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
├── modules/
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── node-group/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── irsa/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── security-groups/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── addons/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── infra/
    └── main.tf (account-level: backend, remote state)
```

---

## Remote State Backend

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "eks/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

---

## VPC Module

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name                                     = "${var.environment}-public-${count.index}"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                 = "1"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name                                       = "${var.environment}-private-${count.index}"
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
  }
}
```

---

## EKS Module

```hcl
# modules/eks/main.tf
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = var.endpoint_private_access
    endpoint_public_access  = var.endpoint_public_access
    public_access_cidrs     = var.public_access_cidrs
  }

  encryption_config {
    provider {
      key_arn = var.kms_key_arn
    }
    resources = ["secrets"]
  }

  enabled_cluster_log_types = var.enabled_cluster_log_types

  tags = {
    Name        = var.cluster_name
    Environment = var.environment
  }
}

resource "aws_eks_addon" "this" {
  for_each = var.addons

  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = each.value.name
  addon_version               = each.value.version
  resolve_conflicts_on_create = "OVERWRITE"
}
```

---

## IAM Module

```hcl
# IAM roles for EKS
resource "aws_iam_role" "eks_cluster" {
  name = "${var.environment}-eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_amazon" {
  role       = aws_iam_role.eks_cluster.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role" "node_group" {
  name = "${var.environment}-eks-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "node_amazon_eks_worker" {
  role       = aws_iam_role.node_group.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "node_amazon_cni" {
  role       = aws_iam_role.node_group.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "node_ecr_read" {
  role       = aws_iam_role.node_group.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}
```

---

## Node Group Module

```hcl
# modules/node-group/main.tf
resource "aws_eks_node_group" "main" {
  cluster_name    = var.cluster_name
  node_group_name = "${var.environment}-${var.node_group_name}"
  node_role_arn   = aws_iam_role.node_role.arn
  version         = var.kubernetes_version
  subnet_ids      = var.subnet_ids
  instance_types  = var.instance_types
  capacity_type   = var.capacity_type
  disk_size       = var.disk_size

  scaling_config {
    desired_size = var.desired_size
    max_size     = var.max_size
    min_size     = var.min_size
  }

  launch_template {
    id      = aws_launch_template.main.id
    version = aws_launch_template.main.latest_version
  }

  tags = {
    "k8s.io/cluster-autoscaler/${var.cluster_name}" = "owned"
  }
}

resource "aws_launch_template" "main" {
  name_prefix = "${var.environment}-${var.node_group_name}"

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size = var.disk_size
      volume_type = "gp3"
      encrypted   = true
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.environment}-${var.node_group_name}"
      Environment = var.environment
    }
  }
}
```

---

## IRSA Module

```hcl
# modules/irsa/main.tf
data "aws_eks_cluster" "main" {
  name = var.cluster_name
}

data "aws_eks_cluster_auth" "main" {
  name = var.cluster_name
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [var.oidc_thumbprint]
  url             = data.aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_role" "sa_role" {
  name = "${var.service_account_name}-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${data.aws_eks_cluster.main.identity[0].oidc[0].issuer}:sub" = "system:serviceaccount:${var.namespace}:${var.service_account_name}"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "sa_policy" {
  role       = aws_iam_role.sa_role.name
  policy_arn = var.policy_arn
}

resource "kubernetes_service_account" "main" {
  metadata {
    name      = var.service_account_name
    namespace = var.namespace
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.sa_role.arn
    }
  }
}
```

---

## Infrastructure Main

```hcl
# environments/production/main.tf
module "vpc" {
  source               = "../../modules/vpc"
  environment          = "production"
  cluster_name         = "eks-production"
  cidr_block           = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20"]
  private_subnet_cidrs = ["10.0.48.0/20", "10.0.64.0/20", "10.0.80.0/20"]
  availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

module "eks" {
  source       = "../../modules/eks"
  cluster_name = "eks-production"
  environment  = "production"
  subnet_ids   = module.vpc.private_subnet_ids
  kubernetes_version = "1.31"

  addons = {
    vpc-cni = {
      name = "vpc-cni"
      version = "v1.18.0-eksbuild.1"
    }
    kube-proxy = {
      name = "kube-proxy"
      version = "v1.31.0-eksbuild.1"
    }
    coredns = {
      name = "coredns"
      version = "v1.11.1-eksbuild.8"
    }
    aws-ebs-csi-driver = {
      name = "aws-ebs-csi-driver"
      version = "v1.34.0-eksbuild.1"
    }
  }
}

module "node_group" {
  source              = "../../modules/node-group"
  cluster_name        = module.eks.cluster_name
  environment         = "production"
  node_group_name     = "general"
  instance_types      = ["m5.large"]
  capacity_type       = "ON_DEMAND"
  desired_size        = 3
  max_size            = 10
  min_size            = 1
  subnet_ids          = module.vpc.private_subnet_ids
  kubernetes_version = "1.31"
}
```

---

## KMS Module

```hcl
# modules/kms/main.tf
resource "aws_kms_key" "eks" {
  description             = "EKS encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_kms_alias" "eks" {
  name          = "alias/eks/${var.environment}"
  target_key_id = aws_kms_key.eks.id
}
```

---

## ECR Module

```hcl
# modules/ecr/main.tf
resource "aws_ecr_repository" "app" {
  name                 = var.repository_name
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key        = aws_kms_key.ecr.arn
  }
}

variable "repository_name" {
  type = string
}
```

---

## Operational Flow

```bash
# Initialize backend
cd environments/production
terraform init

# Validate
terraform validate

# Plan
terraform plan -out=tfplan

# Apply
terraform apply tfplan

# Get kubeconfig
aws eks update-kubeconfig --name eks-production --region us-east-1

# Deploy Kubernetes resources via GitOps (Argo CD)
kubectl apply -f argocd-applications.yaml
```

---

## State & Plan Flow

```
Terraform State (S3) + Lock (DynamoDB)
        │
        ▼
Terraform CLI → Reads config + state → Computes diff
        │
        ▼
Terraform Plan (dry run) → Approval (for prod)
        │
        ▼
Terraform Apply → API calls to AWS (create/update/delete resources)
        │
        ▼
Updated State written back to S3
```

---

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| State | Remote state in S3 with DynamoDB locking |
| Environments | Separate workspaces/directories |
| Modules | Reusable, versioned modules |
| Validation | `terraform validate` + `plan` in CI |
| Drift | `terraform plan` in nightly CI job |
| Secrets | Use provider-level secrets, not in tfvars |
| K8s resources | Use GitOps (Argo CD) rather than Terraform for in-cluster resources |
| Version control | Pin Terraform provider versions |
| Tags | Enforce consistent tagging for Cost Explorer |

---

## References

- [Terraform AWS Provider EKS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_cluster)
- [Terraform AWS VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws)
- [Terraform EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws)
- [AWS EKS Blueprints](https://aws.amazon.com/blogs/containers/announcing-the-amazon-eks-blueprints/)