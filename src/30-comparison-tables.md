# 30. Comparison Tables

## Overview

This section provides concise comparison tables for the most important EKS, Kubernetes, and AWS service decisions.

---

## EKS vs ECS

| Dimension | Amazon EKS | Amazon ECS |
|-----------|------------|------------|
| Orchestration | Kubernetes (open standard) | ECS (AWS proprietary) |
| Portability | Multi-cloud/on-prem | AWS-only |
| Ecosystem | Kubernetes ecosystem (Helm, operators, service mesh) | AWS-native |
| Learning curve | Steeper | Simpler |
| Compute options | Managed node groups, Fargate, Auto Mode, self-managed | Fargate, EC2 |
| Scaling | Karpenter, Cluster Autoscaler, HPA | Service Auto Scaling |
| Load balancing | ALB/NLB (controller) | ALB/NLB (native) |
| IAM | IRSA / Pod Identity | Task role |
| Multi-tenancy | Namespace + RBAC + NetworkPolicy | Less namespace isolation |
| Best for | Complex/multi-cloud, K8s expertise | Simple, AWS-native, fast start |

---

## EKS vs EKS Anywhere

| Dimension | EKS | EKS Anywhere |
|-----------|-----|--------------|
| Location | AWS Cloud | On-premises |
| Control plane | AWS-managed | Customer-managed |
| Management | AWS Console/CLI/API | eksctl |
| Networking | VPC CNI, AWS network | Calico, customer network |
| Storage | EBS, EFS, S3 | Local/SAN |
| Upgrades | AWS handles control plane | Customer upgrades |
| Licensing | Per-cluster pricing | Subscription |
| Use case | Cloud-native | Air-gapped, data sovereignty |

---

## Managed Node Group vs Self-Managed Node

| Dimension | Managed Node Group | Self-Managed Node |
|-----------|--------------------|-------------------|
| AMI | EKS-optimized (auto-updated) | Customer-provided |
| ASG | AWS-managed | Customer-managed |
| Scaling | AWS-managed | Customer |
| Customization | Limited (launch template) | Full |
| kubelet | AWS-managed | Customer |
| Bootstrap | AWS-provided | Customer script |
| Patching | Easier (node group update) | Manual |
| Best for | General workloads | Custom AMI/hardware |

---

## Managed Node Group vs Fargate

| Dimension | Managed Node Group | Fargate |
|-----------|--------------------|---------|
| Management | AWS manages ASG/AMI | Fully serverless |
| Compute model | EC2 instances | Per-pod microVM |
| Scaling | ASG (nodes) | Per-pod |
| Customization | Launch templates | Limited |
| DaemonSet support | Yes | No |
| GPU support | Yes | No |
| Host networking | Yes | No |
| Storage | EBS + EFS | EFS only |
| Cost | EC2 pricing (fixed) | Per-pod vCPU/mem |
| Best for | Stateful, GPU, general | Stateless, batch, serverless |

---

## Managed Node Group vs EKS Auto Mode

| Dimension | Managed Node Group | EKS Auto Mode |
|-----------|--------------------|---------------|
| Node provisioning | Customer-defined | AWS automatic |
| Instance selection | Customer | AWS optimizes |
| Scaling | ASG min/max/desired | Auto |
| Storage | Customer-managed CSI | Auto EBS provisioning |
| Networking | Customer-managed CNI | Auto-managed |
| Custom AMIs | Yes (custom) | No |
| Instance placement | Customer control | AWS control |
| Cost visibility | Full EC2 | Less granular |
| Best for | Custom/GPU workloads | Simplified ops, general workloads |

---

## ALB vs NLB

| Dimension | ALB | NLB |
|-----------|-----|-----|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP) |
| Static IP | No | Yes |
| Path routing | Yes | No |
| Host routing | Yes | No |
| WAF | Yes | No |
| WebSocket | Yes | Yes |
| Health checks | HTTP/TCP | TCP/HTTP |
| Target types | IP, Instance | IP, Instance |
| mTLS | Optional (client cert) | TLS passthrough |
| Typical use | Microservices, ingress | TCP apps, static IP, gaming |

---

## EBS vs EFS vs S3

| Dimension | EBS | EFS | S3 |
|-----------|-----|-----|-----|
| Type | Block | File | Object |
| Access mode | ReadWriteOnce | ReadWriteMany | ReadWriteMany |
| Availability | Single AZ | Multi-AZ | Multi-Region |
| Performance | High IOPS (gp3/io2) | Bursting/provisioned | Scale-bound |
| Encryption | Yes | Yes | Yes |
| Snapshots | Yes | Yes | Versioning |
| Cost basis | GB + IOPS | GB + requests | Storage + requests + egress |
| Best for | Databases | Shared media, NFS | Backups, static, object data |

---

## IRSA vs EKS Pod Identity

| Dimension | IRSA | EKS Pod Identity |
|-----------|------|------------------|
| Provider | OIDC | EKS service principal |
| Setup complexity | Moderate | Simple |
| Trust policy | Customer-managed | AWS-managed |
| Multi-tenancy | Per-SA roles | Native |
| Region support | All | Limited (expanding) |
| Agent | None (webhook) | eks-pod-identity-agent |
| Best for | Existing clusters | New deployments |

---

## HPA vs VPA

| Dimension | HPA | VPA |
|-----------|-----|-----|
| What it scales | Number of replicas | Resource requests/limits |
| Metric source | CPU, memory, custom, external | Historical metrics |
| Mode | Apply immediately | Off/Initial/Auto |
| Eviction | Creates new pods | May restart pods |
| Use case | Burstable workloads | Stable, right-sizing |
| Combined use | Yes (with VPA recommendations) | Yes (with HPA) |

---

## Cluster Autoscaler vs Karpenter

| Dimension | Cluster Autoscaler | Karpenter |
|-----------|--------------------|-----------|
| Provisioning | Via ASG | Direct EC2 |
| Scale-up speed | 5-10 min | ~60 sec |
| Instance flexibility | ASG-defined | Any instance type |
| Spot | Via ASG | Native |
| Consolidation | No | Yes |
| Node selection | Limited | Highly flexible |
| Overhead | ASG | Lower |
| Best for | Predictable workloads | Dynamic, cost-optimized |

---

## Secrets Manager vs Parameter Store vs Kubernetes Secrets

| Dimension | Secrets Manager | SSM Parameter Store | K8s Secrets |
|-----------|-----------------|---------------------|-------------|
| Encryption | KMS | KMS (SecureString) | etcd + KMS |
| Rotation | Automatic (Lambda) | Manual | Manual |
| Cost | $0.40/secret/month | $0.05/param/month | Free |
| Integration | External Secrets/CSI | External Secrets/CSI | Native |
| Use case | App secrets, rotation | Config values | Short-lived, namespaced |

---

## CloudWatch vs Prometheus vs Grafana

| Dimension | CloudWatch | Prometheus | Grafana |
|-----------|------------|------------|---------|
| Purpose | AWS monitoring/logs | Metrics time-series | Visualization/dashboards |
| Data model | AWS-native metrics | Pull-based metrics | Frontend for metrics sources |
| Multicloud | Limited | Yes | Yes |
| Cost | Per metrics/logs | Self-hosted (or AMP) | Free (or AMG) |
| Best for | AWS-native monitoring | K8s monitoring | Dashboards across sources |
| Combined | — | CloudWatch + Prometheus feed | Grafana visualizes both |

---

## Terraform vs CloudFormation

| Dimension | Terraform | CloudFormation |
|-----------|-----------|----------------|
| Language | HCL | YAML/JSON |
| Vendor | HashiCorp (multi-cloud) | AWS native |
| State | External (S3/remote) | AWS-managed (stack) |
| Resource types | Any provider | AWS + some third-party |
| Modularity | Strong modules | Nested stacks |
| Drift detection | Manual `plan` | Drift detection |
| Best for | Multi-cloud, IaC pervasive | AWS-only teams |

---

## Summary Decision Matrix

| Decision | Recommendation |
|----------|----------------|
| Compute model | Managed Node Groups + Spot; Auto Mode for SMB; Fargate for serverless |
| Storage | gp3 EBS for stateful; EFS for shared; S3 for objects |
| Pod IAM | EKS Pod Identity (new); IRSA for existing |
| Autoscaling | Karpenter for cluster; HPA for pods; VPA for right-sizing; KEDA for event-driven |
| Secrets | Secrets Manager/SSM + External Secrets |
| Monitoring | Prometheus + Grafana + CloudWatch |
| IaC | Terraform (multi-cloud) or CloudFormation (AWS-only) |
| CI/CD | GitHub Actions + Argo CD (GitOps) |

---

## References

- [EKS Compute Options](https://docs.aws.amazon.com/eks/latest/userguide/eks-compute.html)
- [EKS Pod Identity vs IRSA](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [Karpenter docs](https://karpenter.sh/docs/)
- [EKS Best Practices](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)