# Amazon EKS Deep-Dive Documentation

A comprehensive, enterprise-grade knowledge base covering Amazon EKS architecture, operations, security, and production best practices.

## Structure

Each file follows this consistent format: **Concept → Architecture → Components → How It Works → Configuration → Example → Production Best Practices → Security → Cost → Troubleshooting → References**.

Components are categorized as **AWS-managed**, **Kubernetes-native**, **add-ons**, or **third-party**.

## Table of Contents

| File | Topic |
|------|-------|
| [01-executive-overview.md](01-executive-overview.md) | Executive overview, benefits, use cases |
| [02-core-architecture.md](02-core-architecture.md) | EKS control plane, data plane, VPC, traffic flows |
| [03-cluster-types-compute.md](03-cluster-types-compute.md) | Managed node groups, Auto Mode, EKS Capabilities, Fargate, self-managed, Graviton |
| [04-networking-deep-dive.md](04-networking-deep-dive.md) | VPC design, CNI, prefix delegation, services, ingress |
| [05-iam-security.md](05-iam-security.md) | IAM, Access Entries, RBAC, Pod Identity, IRSA |
| [06-add-ons.md](06-add-ons.md) | CoreDNS, kube-proxy, VPC CNI, EBS/EFS CSI, LB controller |
| [07-storage-deep-dive.md](07-storage-deep-dive.md) | EBS, EFS, S3 decision matrix, PVCs, snapshots, backup |
| [08-scaling.md](08-scaling.md) | HPA, VPA, KEDA, Cluster Autoscaler, Karpenter, PDB |
| [09-ingress-application-traffic.md](09-ingress-application-traffic.md) | Traffic flow: DNS → CDN → WAF → ALB → Ingress → Service → Pod |
| [10-observability.md](10-observability.md) | Metrics, logs, traces, Container Insights, Prometheus, ADOT |
| [11-monitoring-alerting.md](11-monitoring-alerting.md) | CloudWatch alarms, Prometheus rules, alert routing |
| [12-logging-architecture.md](12-logging-architecture.md) | Fluent Bit pipeline, structured logging, retention |
| [13-control-plane-logging.md](13-control-plane-logging.md) | api/audit/authenticator/controllerManager/scheduler logs |
| [14-deployment-cicd.md](14-deployment-cicd.md) | GitHub Actions, Helm, Kustomize, Argo CD, blue/green/canary |
| [15-container-registry-ecr.md](15-container-registry-ecr.md) | ECR lifecycle, scanning, replication, pull-through cache |
| [16-security-scanning-devsecops.md](16-security-scanning-devsecops.md) | Inspector, GuardDuty, Security Hub, Trivy, Falco, CodeQL |
| [17-secrets-management.md](17-secrets-management.md) | Secrets Manager, SSM, Secrets Store CSI, External Secrets |
| [18-encryption.md](18-encryption.md) | KMS envelope encryption, EBS/EFS/S3/ECR encryption, TLS |
| [19-high-availability.md](19-high-availability.md) | Multi-AZ, anti-affinity, PDB, failure scenarios |
| [20-disaster-recovery.md](20-disaster-recovery.md) | Velero, EBS snapshots, AWS Backup, RTO/RPO |
| [21-upgrades.md](21-upgrades.md) | Upgrade process, pre-checks, add-ons, nodes, rollback |
| [22-extended-support.md](22-extended-support.md) | Standard vs extended support lifecycle, pricing |
| [23-cost-optimization.md](23-cost-optimization.md) | Cost model, Spot, Savings Plans, NAT/storage strategies |
| [24-terraform.md](24-terraform.md) | Terraform IaC, modules, remote state, IRSA, KMS |
| [25-kubernetes-resources.md](25-kubernetes-resources.md) | Namespace → Deployment → Services → HPA resource primer |
| [26-application-deployment-example.md](26-application-deployment-example.md) | Full app deployment example (frontend/backend/DB) |
| [27-troubleshooting.md](27-troubleshooting.md) | Cluster/node/pod/networking/IAM/storage/Helm troubleshooting |
| [28-production-best-practices.md](28-production-best-practices.md) | Security, reliability, performance, operations, cost |
| [29-enterprise-reference-architecture.md](29-enterprise-reference-architecture.md) | Full enterprise architecture with all services |
| [30-comparison-tables.md](30-comparison-tables.md) | EKS vs ECS, ALB vs NLB, EBS vs EFS vs S3, and more |
| [31-sops.md](31-sops.md) | 20 standard operating procedures |
| [32-command-reference.md](32-command-reference.md) | AWS CLI, kubectl, helm, eksctl, terraform, docker cheatsheet |
| [33-glossary.md](33-glossary.md) | EKS / Kubernetes / AWS terminology |
| [34-production-readiness-checklist.md](34-production-readiness-checklist.md) | Pre-launch and periodic review checklist |
| [35-references.md](35-references.md) | Curated official AWS + Kubernetes references |
| [36-eks-workshop-guide.md](36-eks-workshop-guide.md) | Hands-on lab guide: workshop paths + module mapping |

## Notes

- Primary references are official AWS docs; see [35-references.md](35-references.md).
- Where AWS offerings are deprecated or legacy, this guide flags them (e.g., `aws-auth` replaced by Access Entries).
- Workloads are categorized by who manages them: AWS-managed, Kubernetes-native, add-on, or third-party.

## Suggested Reading Order

1. **Foundation**: 01 → 02 → 03
2. **Networking**: 04 → 09
3. **Security**: 05 → 17 → 18
4. **Compute & Scaling**: 03 → 08
5. **Storage**: 07
6. **Operations**: 06 → 21 → 22 → 27 → 32
7. **Production**: 10–13 → 19 → 20 → 23 → 28 → 34