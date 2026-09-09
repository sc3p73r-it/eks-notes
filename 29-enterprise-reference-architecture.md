# 29. Enterprise Reference Architecture

## Overview

This section presents a complete enterprise reference architecture incorporating all major AWS services with EKS. It includes the architecture diagram, network flow, security flow, deployment flow, monitoring flow, and backup/DR flow.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                  ENTERPRISE AWS ARCHITECTURE                               │
│                                                                                           │
│  ┌──────────────────────────┐     ┌──────────────────────────┐                           │
│  │      Route 53            │     │       CloudFront          │                           │
│  │  (DNS, failover)         │────▶│  (CDN, TLS, WAF)         │                           │
│  └──────────────────────────┘     └────────────┬─────────────┘                           │
│                                                │                                          │
│                                                ▼                                          │
│                                     ┌──────────────────────────┐                          │
│                                     │       AWS WAF             │                          │
│                                     └────────────┬─────────────┘                          │
│                                                │                                          │
│                             ┌──────────────────┴──────────────────┐                       │
│                             ▼                                     ▼                       │
│                  ┌──────────────────────────┐          ┌──────────────────────────┐       │
│                  │   Public ALB (Layer 7)   │          │   Internal NLB/ALB       │       │
│                  │   + ACM Certificate      │          │   (backend services)     │       │
│                  └────────────┬─────────────┘          └────────────┬─────────────┘       │
│                                                                                          │
│  ┌─────────────────────────────────┴────────────────────────────────────────────────┐    │
│  │                              VPC (10.0.0.0/16)                                   │    │
│  │                                                                                  │    │
│  │  ┌──────────────────────────────┐   ┌──────────────────────────────┐            │    │
│  │  │  Public Subnets (3 AZ)       │   │  Private Subnets (3 AZ)       │            │    │
│  │  │  ┌────────────────────┐     │   │  ┌─────────────────────────┐ │            │    │
│  │  │  │ ALB                │     │   │  │ EKS Cluster               │ │            │    │
│  │  │  └────────────────────┘     │   │  │ ┌─────────────────────────┐│ │            │    │
│  │  │  ┌────────────────────┐     │   │  │ │  Managed Node Groups    ││ │            │    │
│  │  │  │ NAT Gateway         │     │   │  │  ┌────────────────────┐  ││ │            │    │
│  │  │  └────────────────────┘     │   │  │  │  Pods               │  ││ │            │    │
│  │  └──────────────────────────────┘   │  │  └────────────────────┘  ││ │            │    │
│  │                                     │  └─────────────────────────┘│ │            │    │
│  │                                     │  ┌─────────────────────────┐ │            │    │
│  │                                     │  │  RDS (Aurora/PostgreSQL)│ │            │    │
│  │                                     │  └─────────────────────────┘ │            │    │
│  │                                     │  ┌─────────────────────────┐ │            │    │
│  │                                     │  │  ElastiCache (Redis)     │ │            │    │
│  │                                     │  └─────────────────────────┘ │            │    │
│  │                                     │  ┌─────────────────────────┐ │            │    │
│  │                                     │  │  VPC Endpoints           │ │            │    │
│  │                                     │  │  (S3, ECR, STS, KMS, SSM)│ │            │    │
│  │                                     │  └─────────────────────────┘ │            │    │
│  │  ┌────────────────────────────────────────────────────────────────┐  │            │    │
│  │  │  AWS Services Integration                                      │  │            │    │
│  │  │  ┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐  │  │            │    │
│  │  │  │ ECR    ││ EBS    ││ EFS    ││  S3    ││Secrets ││  KMS   │  │  │            │    │
│  │  │  │        ││        ││        ││        ││Manager ││        │  │  │            │    │
│  │  │  └────────┘└────────┘└────────┘└────────┘└────────┘└────────┘  │  │            │    │
│  │  └────────────────────────────────────────────────────────────────┘  │            │    │
│  └──────────────────────────────────────────────────────────────────┴────────────┘    │
│                                                                                          │
│  ┌──────────────────────────┐    ┌──────────────────────────┐    ┌────────────────────┐  │
│  │  CloudWatch + Container  │    │  GuardDuty + Security Hub│    │  GitHub Actions +   │  │
│  │  Insights + Grafana      │    │  (security posture)      │    │  Terraform + Argo CD│  │
│  └──────────────────────────┘    └──────────────────────────┘    └────────────────────┘  │
│                                                                                          │
│  ┌──────────────────────────┐    ┌──────────────────────────┐                           │
│  │  IAM + Access Control    │    │  CloudTrail (audit)      │                           │
│  └──────────────────────────┘    └──────────────────────────┘                           │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

| Component | Purpose | Managed By |
|-----------|---------|------------|
| Route 53 | DNS, failover routing | AWS |
| CloudFront | CDN, edge caching, TLS | AWS |
| WAF | Web application firewall | AWS |
| ACM | TLS certificates | AWS |
| ALB/NLB | Load balancing | AWS |
| VPC | Networking | Customer |
| EKS | Kubernetes platform | AWS + Customer |
| ECR | Container registry | AWS |
| RDS | Managed database | AWS |
| ElastiCache | Managed Redis | AWS |
| S3 | Object storage | AWS |
| EBS/EFS | Block/file storage | AWS |
| Secrets Manager | Secret storage | AWS |
| KMS | Encryption keys | AWS |
| CloudWatch | Monitoring/logging | AWS |
| CloudTrail | API audit | AWS |
| GuardDuty | Threat detection | AWS |
| Security Hub | Security posture | AWS |
| IAM | Identity and access | AWS |
| GitHub Actions | CI/CD | Third-party |
| Terraform | Infrastructure as Code | Third-party |
| Argo CD | GitOps | Third-party |

---

## Network Flow

```
Internet
  → Route 53 (DNS resolution, failover/alias)
  → CloudFront (CDN cache, TLS termination at edge)
  → AWS WAF (web ACL, rate limiting, managed rules)
  → ALB (public, Layer 7, target group of pod IPs)
  → AWS Load Balancer Controller (maps Ingress → ALB)
  → Ingress resource
  → Kubernetes Service
  → Pod

Internal traffic:
  Pod → API → Internal NLB/Service
  Pod → RDS (security group allow)
  Pod → ElastiCache (security group allow)
  Pod → S3/ECR/STS/KMS via VPC Endpoints
```

---

## Security Flow

```
User IAM Role
  → EKS Access Entry (cluster access)
  → Kubernetes RBAC (namespace-scoped Role)
  → Pod Security Admission
  → NetworkPolicy (pod-level isolation)
  → Runtime: GuardDuty + Falco

Secrets:
  Pod → Pod Identity → Secrets Manager (IAM role)
  → Secret injected as env / mounted via External Secrets

Images:
  Build → Scan (Trivy) → Push to ECR (scanOnPush)
  → Amazon Inspector enhanced scan
  → Deploy only if no critical/high findings
```

---

## Application Deployment Flow

```
Developer commits code
  → GitHub / GitHub Actions triggers pipeline
  → SAST (CodeQL/SonarQube)
  → Build Docker image
  → Scan (Trivy)
  → Push to ECR
  → Update GitOps repo (tag)
  → Argo CD detects change
  → Syncs to EKS (canary/blue-green via Argo Rollouts)
  → Kubernetes creates new Deployment
  → Rolling update/Rollout
  → Health checks (readiness/liveness)
  → Traffic cutover
```

---

## Monitoring Flow

```
EKS Cluster
  → CloudWatch Container Insights (metrics + logs)
  → Prometheus (kube-prometheus-stack) → custom metrics
  → CloudWatch Logs (control plane + container logs via Fluent Bit)
  → X-Ray / OpenTelemetry (traces)
  → Grafana (dashboards)
  → Prometheus AlertManager + CloudWatch Alarms
  → Slack / PagerDuty / Opsgenie
  → Guards: CloudTrail + GuardDuty + Security Hub
```

---

## Backup / DR Flow

```
Primary Cluster (us-east-1)
  → Velero (cluster resources → S3)
  → EBS snapshots (stateful volumes)
  → AWS Backup (RDS/EFS)
  → ECR cross-region replication
  → Terraform state (S3 backend + locking)

DR Cluster (us-west-2)
  → Warm admission / pilot light
  → Argo CD syncs manifests from Git
  → Restore data from snapshots
  → Route 53 failover
```

---

## Key Production Recommendations

| Area | Recommendation |
|------|----------------|
| Networking | Use private API endpoint, restrict public access CIDRs |
| Security | Least-privilege IAM, Pod Security restricted, NetworkPolicy default-deny |
| Compute | Managed node groups or EKS Auto Mode; spot for stateless |
| Storage | gp3 EBS for stateful, EFS for shared, S3 for objects |
| Databases | Use RDS/Aurora Multi-AZ, not in-cluster PostgreSQL for production |
| CI/CD | GitOps with Argo CD + immutable image tags (git SHA) |
| Monitoring | Container Insights + Prometheus + X-Ray |
| DR | Pilot light cluster in secondary region, test quarterly |
| Cost | Compute Savings Plan for base, spot + scalability |

---

## References

- [AWS Well-Architected for Kubernetes](https://docs.aws.amazon.com/wellarchitected/latest/kubernetes-workloads/welcome.html)
- [EKS Reference Architecture](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [EKS Blueprints](https://aws.amazon.com/blogs/containers/announcing-the-amazon-eks-blueprints/)