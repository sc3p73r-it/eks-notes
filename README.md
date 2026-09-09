# Amazon EKS Deep-Dive Documentation

A comprehensive, enterprise-grade knowledge base covering Amazon EKS architecture, operations, security, and production best practices.

This repo is also published as a **book-style website** on GitHub Pages (built with mdBook). Topic files live in [`src/`](src/); the chapter order is defined in [`src/SUMMARY.md`](src/SUMMARY.md).

## Table of Contents

| File | Topic |
|------|-------|
| [src/01-executive-overview.md](src/01-executive-overview.md) | Executive overview, benefits, use cases |
| [src/02-core-architecture.md](src/02-core-architecture.md) | EKS control plane, data plane, VPC, traffic flows |
| [src/03-cluster-types-compute.md](src/03-cluster-types-compute.md) | Managed node groups, Auto Mode, EKS Capabilities, Fargate, self-managed, Graviton |
| [src/04-networking-deep-dive.md](src/04-networking-deep-dive.md) | VPC design, CNI, prefix delegation, services, ingress |
| [src/05-iam-security.md](src/05-iam-security.md) | IAM, Access Entries, RBAC, Pod Identity, IRSA |
| [src/06-add-ons.md](src/06-add-ons.md) | CoreDNS, kube-proxy, VPC CNI, EBS/EFS CSI, LB controller |
| [src/07-storage-deep-dive.md](src/07-storage-deep-dive.md) | EBS, EFS, S3 decision matrix, PVCs, snapshots, backup |
| [src/08-scaling.md](src/08-scaling.md) | HPA, VPA, KEDA, Cluster Autoscaler, Karpenter, PDB |
| [src/09-ingress-application-traffic.md](src/09-ingress-application-traffic.md) | Traffic flow: DNS → CDN → WAF → ALB → Ingress → Service → Pod |
| [src/10-observability.md](src/10-observability.md) | Metrics, logs, traces, Container Insights, Prometheus, ADOT |
| [src/11-monitoring-alerting.md](src/11-monitoring-alerting.md) | CloudWatch alarms, Prometheus rules, alert routing |
| [src/12-logging-architecture.md](src/12-logging-architecture.md) | Fluent Bit pipeline, structured logging, retention |
| [src/13-control-plane-logging.md](src/13-control-plane-logging.md) | api/audit/authenticator/controllerManager/scheduler logs |
| [src/14-deployment-cicd.md](src/14-deployment-cicd.md) | GitHub Actions, Helm, Kustomize, Argo CD, blue/green/canary |
| [src/15-container-registry-ecr.md](src/15-container-registry-ecr.md) | ECR lifecycle, scanning, replication, pull-through cache |
| [src/16-security-scanning-devsecops.md](src/16-security-scanning-devsecops.md) | Inspector, GuardDuty, Security Hub, Trivy, Falco, CodeQL |
| [src/17-secrets-management.md](src/17-secrets-management.md) | Secrets Manager, SSM, Secrets Store CSI, External Secrets |
| [src/18-encryption.md](src/18-encryption.md) | KMS envelope encryption, EBS/EFS/S3/ECR encryption, TLS |
| [src/19-high-availability.md](src/19-high-availability.md) | Multi-AZ, anti-affinity, PDB, failure scenarios |
| [src/20-disaster-recovery.md](src/20-disaster-recovery.md) | Velero, EBS snapshots, AWS Backup, RTO/RPO |
| [src/21-upgrades.md](src/21-upgrades.md) | Upgrade process, pre-checks, add-ons, nodes, rollback |
| [src/22-extended-support.md](src/22-extended-support.md) | Standard vs extended support lifecycle, pricing |
| [src/23-cost-optimization.md](src/23-cost-optimization.md) | Cost model, Spot, Savings Plans, NAT/storage strategies |
| [src/24-terraform.md](src/24-terraform.md) | Terraform IaC, modules, remote state, IRSA, KMS |
| [src/25-kubernetes-resources.md](src/25-kubernetes-resources.md) | Namespace → Deployment → Services → HPA resource primer |
| [src/26-application-deployment-example.md](src/26-application-deployment-example.md) | Full app deployment example (frontend/backend/DB) |
| [src/27-troubleshooting.md](src/27-troubleshooting.md) | Cluster/node/pod/networking/IAM/storage/Helm troubleshooting |
| [src/28-production-best-practices.md](src/28-production-best-practices.md) | Security, reliability, performance, operations, cost |
| [src/29-enterprise-reference-architecture.md](src/29-enterprise-reference-architecture.md) | Full enterprise architecture with all services |
| [src/30-comparison-tables.md](src/30-comparison-tables.md) | EKS vs ECS, ALB vs NLB, EBS vs EFS vs S3, and more |
| [src/31-sops.md](src/31-sops.md) | 20 standard operating procedures |
| [src/32-command-reference.md](src/32-command-reference.md) | AWS CLI, kubectl, helm, eksctl, terraform, docker cheatsheet |
| [src/33-glossary.md](src/33-glossary.md) | EKS / Kubernetes / AWS terminology |
| [src/34-production-readiness-checklist.md](src/34-production-readiness-checklist.md) | Pre-launch and periodic review checklist |
| [src/35-references.md](src/35-references.md) | Curated official AWS + Kubernetes references |
| [src/36-eks-workshop-guide.md](src/36-eks-workshop-guide.md) | Hands-on lab guide: workshop paths + module mapping |
| [src/37-eks-architecture-diagram.md](src/37-eks-architecture-diagram.md) | EKS architecture diagram |

## Book Site

- Local preview: `mdbook serve` (opens http://localhost:3000)
- Build output: `mdbook build` → `book/` directory
- GitHub Pages deployment: `.github/workflows/pages.yml` builds and publishes on every push to `main`

## Notes

- Primary references are official AWS docs; see [35-references.md](src/35-references.md).
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