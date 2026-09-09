# 1. Executive Overview

## What Is Amazon EKS?

Amazon Elastic Kubernetes Service (Amazon EKS) is a fully managed Kubernetes service that runs Kubernetes control plane and worker nodes across multiple AWS Availability Zones. AWS operates the Kubernetes control plane—API servers, etcd, scheduler, and controller manager—so that customers can focus on deploying and operating containerized workloads on the data plane.

EKS is certified Kubernetes-conformant, meaning workloads that run on upstream Kubernetes run identically on EKS without modification. This portability is a fundamental design principle: organizations can migrate workloads between on-premises Kubernetes, EKS, and other cloud-provider Kubernetes offerings without refactoring.

AWS provides two primary modes of operating EKS:

- **EKS Standard**: AWS manages the control plane; the customer manages the data plane (nodes, pods, networking, storage).
- **EKS Auto Mode**: AWS manages both the control plane and data plane, including compute provisioning, node lifecycle, storage, and networking.

---

## Why Organizations Use EKS

| Reason | Explanation |
|--------|-------------|
| **Operational burden reduction** | AWS patches, scales, and maintains the control plane; customers focus on workloads |
| **Compliance** | EKS is SOC, HIPAA, PCI DSS, FedRAMP, and ISO compliant out of the box |
| **AWS ecosystem integration** | Native integration with IAM, VPC, ECR, CloudWatch, Secrets Manager, KMS, ALB/NLB, and dozens of other AWS services |
| **Kubernetes portability** | Certified Kubernetes-conformant; workloads are portable across environments |
| **Scalability** | Leverages AWS Auto Scaling, Karpenter, and managed node groups for elastic infrastructure |
| **Security** | IRSA, Pod Identity, encryption at rest, private clusters, GuardDuty integration |
| **Multi-tenancy** | Namespace-based isolation with RBAC, NetworkPolicy, and resource quotas |
| **Hybrid and edge** | EKS Anywhere for on-premises; EKS Hybrid Nodes for connecting existing infrastructure |

---

## High-Level EKS Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          AWS Region (us-east-1)                         │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                AWS-Managed Account (Control Plane)               │   │
│  │                                                                  │   │
│  │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │   │
│  │   │   AZ-1a     │   │   AZ-1b     │   │   AZ-1c     │          │   │
│  │   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │          │   │
│  │   │  │ API   │  │   │  │ API   │  │   │  │ API   │  │          │   │
│  │   │  │Server │  │   │  │Server │  │   │  │Server │  │          │   │
│  │   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │          │   │
│  │   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │          │   │
│  │   │  │ etcd  │  │   │  │ etcd  │  │   │  │ etcd  │  │          │   │
│  │   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │          │   │
│  │   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │          │   │
│  │   │  │Sched. │  │   │  │Sched. │  │   │  │Sched. │  │          │   │
│  │   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │          │   │
│  │   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │          │   │
│  │   │  │CtrlMgr│  │   │  │CtrlMgr│  │   │  │CtrlMgr│  │          │   │
│  │   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │          │   │
│  │   └─────────────┘   └─────────────┘   └─────────────┘          │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                    ┌───────────────┴───────────────┐                    │
│                    │   Customer VPC (10.0.0.0/16)  │                    │
│                    │                                │                    │
│                    │  ┌──────────┐  ┌──────────┐   │                    │
│                    │  │ Public   │  │ Private  │   │                    │
│                    │  │ Subnet   │  │ Subnet   │   │                    │
│                    │  │ ┌──────┐ │  │ ┌──────┐ │   │                    │
│                    │  │ │ALB   │ │  │ │Node1 │ │   │                    │
│                    │  │ └──────┘ │  │ │Node2 │ │   │                    │
│                    │  └──────────┘  │ └──────┘ │   │                    │
│                    │                └──────────┘   │                    │
│                    └────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## EKS vs Self-Managed Kubernetes

| Dimension | Amazon EKS | Self-Managed Kubernetes |
|-----------|------------|------------------------|
| Control plane | AWS-managed, HA across 3 AZs | Customer-managed, must build HA |
| etcd | Managed, encrypted, backed up | Customer installs, operates, backs up |
| Patching | AWS applies security patches automatically | Customer must patch manually |
| Upgrades | Managed upgrade path with documented process | Full manual upgrade process |
| IAM | Native AWS IAM integration | Must configure OIDC or webhook manually |
| Networking | VPC CNI (VPC-native) | Any CNI; must configure manually |
| Load balancing | AWS Load Balancer Controller (ALB/NLB) | Must install and configure manually |
| Monitoring | CloudWatch, Container Insights | Must deploy Prometheus, Grafana, etc. |
| Compliance | SOC, HIPAA, PCI DSS, FedRAMP | Must achieve and maintain independently |
| Cost | $0.10/hr control plane + EC2/data costs | EC2 costs only, but higher operational cost |
| Operational burden | Low (AWS manages control plane) | High (full Kubernetes lifecycle) |
| Customization | High (any K8s add-on, any CNI) | Unlimited |
| Version lag | Typically lags upstream by 1-2 versions | Can run any version immediately |

---

## EKS vs EKS Anywhere

| Dimension | Amazon EKS | EKS Anywhere |
|-----------|------------|--------------|
| Deployment | AWS Cloud | On-premises / customer data center |
| Control plane | AWS-managed | Customer-managed (runs on customer infra) |
| Management | AWS Console, CLI, API | eksctl CLI |
| Licensing | EKS cluster pricing | Subscription-based |
| Support | AWS Support | AWS EKS Anywhere support |
| Hardware | AWS EC2 / Fargate | Customer-provided servers |
| Networking | VPC CNI, AWS networking | Calico CNI, customer networking |
| Storage | EBS, EFS, S3 | Local storage, customer SAN |
| Upgrades | AWS-managed control plane upgrade | Customer upgrades both planes |
| Use case | Cloud-native, AWS workloads | Air-gapped, data sovereignty, edge |

---

## EKS vs ECS

| Dimension | Amazon EKS | Amazon ECS |
|-----------|------------|------------|
| Orchestration | Kubernetes (open standard) | ECS (AWS proprietary) |
| Portability | Portable across clouds and on-prem | AWS-only |
| Ecosystem | Massive Kubernetes ecosystem (Helm, Argo CD, Istio, etc.) | AWS-native tooling |
| Learning curve | Steeper (Kubernetes concepts) | Simpler (fewer concepts) |
| Configuration | Extensive (CRDs, operators, admission controllers) | More opinionated, less extensible |
| Scaling | Karpenter, Cluster Autoscaler, HPA | Service Auto Scaling, Capacity Providers |
| Load balancing | ALB/NLB via AWS LB Controller | ALB/NLB via ECS integration |
| IAM | IRSA, Pod Identity (granular per pod) | Task-level IAM roles |
| Fargate | Supported (per-pod serverless) | Native (primary compute option) |
| Service discovery | CoreDNS, K8s Service | Cloud Map |
| Networking | VPC CNI (VPC-native) | awsvpc (ENI per task) |
| Rolling updates | Native K8s deployment strategy | Deployment controller (rolling, blue/green) |
| Custom resources | CRDs, operators | Limited to AWS resources |
| Best for | Multi-cloud, Kubernetes expertise, complex workloads | Simple deployments, AWS-native, faster time-to-value |

---

## EKS Benefits

1. **Managed Control Plane**: AWS operates, scales, patches, and backs up the Kubernetes control plane across three Availability Zones with automatic failover.
2. **Security by Default**: Encryption at rest (KMS), private API server endpoint, integration with IAM, GuardDuty, and Security Hub.
3. **AWS Ecosystem Integration**: Native integration with 80+ AWS services including IAM, VPC, ECR, Secrets Manager, KMS, ALB, CloudWatch, and more.
4. **High Availability**: Multi-AZ control plane, managed node groups across AZs, and built-in resilience.
5. **Compliance**: SOC 1/2/3, HIPAA, PCI DSS Level 1, FedRAMP, ISO 27001, and more.
6. **Cost Efficiency**: No charge for the control plane (only $0.10/hr per cluster), plus EC2/data costs. Use Spot Instances for up to 90% savings.
7. **Kubernetes Conformity**: Certified Kubernetes-compatible; workloads are portable.
8. **Hybrid Support**: EKS Anywhere for on-premises, EKS Hybrid Nodes for connecting existing infrastructure.
9. **Observability**: Native CloudWatch integration, Container Insights, Prometheus, Grafana support.
10. **Auto Scaling**: Karpenter for intelligent node provisioning, HPA for pod scaling.

---

## EKS Limitations

| Limitation | Impact |
|------------|--------|
| Control plane access | Cannot SSH into control plane; limited control plane visibility |
| Version lag | EKS typically supports N-2 to N+1 Kubernetes versions; may lag upstream |
| Regional availability | Not available in all AWS regions |
| etcd management | Cannot directly access or manage etcd; no custom etcd configuration |
| Control plane customization | Cannot modify control plane component flags or configuration |
| Add-on compatibility | Some add-ons may require specific versions or configurations |
| Networking | VPC CNI is the default; custom CNIs require additional configuration |
| Cost | $0.10/hr per cluster (~$73/month) plus EC2, data transfer, and storage costs |
| Upgrade timeline | Must upgrade within supported version window or lose support |
| API rate limits | Kubernetes API has request rate limits; large clusters need careful management |

---

## Typical Enterprise EKS Use Cases

| Use Case | Description |
|----------|-------------|
| **Microservices architecture** | Decompose monoliths into independently deployable services |
| **CI/CD platform** | Run Jenkins, GitLab runners, Argo Workflows on EKS |
| **ML/AI pipelines** | Run Kubeflow, SageMaker operators, training jobs on GPU instances |
| **Data processing** | Run Apache Spark, Flink, Airflow on Kubernetes |
| **Hybrid cloud** | Connect on-premises workloads with EKS via VPN/Direct Connect |
| **Multi-region deployment** | Deploy identical clusters across regions for DR and low latency |
| **Platform engineering** | Build internal developer platforms with Backstage, Crossplane |
| **SaaS platforms** | Multi-tenant SaaS with namespace isolation |
| **Edge computing** | EKS Hybrid Nodes for edge locations, EKS Anywhere for air-gapped |
| **Legacy modernization** | Containerize and orchestrate legacy applications |

---

## When EKS Is and Is Not Appropriate

### When EKS Is Appropriate

- Organization has Kubernetes expertise or plans to invest in it
- Workloads need to be portable across clouds or on-premises
- Complex microservices architecture with many services
- Need for advanced scheduling (GPU, custom resources, operators)
- Multi-tenancy requirements with strong isolation
- Regulatory compliance requiring specific infrastructure controls
- Hybrid or multi-cloud strategy

### When EKS Is Not Appropriate

- Simple application with few services (ECS may be simpler)
- Team has no Kubernetes experience and limited time to learn
- Very small workloads where the $0.10/hr cluster cost is disproportionate
- Workloads that are purely AWS-native with no portability requirement
- Need for immediate access to latest Kubernetes features (version lag)
- Extremely latency-sensitive workloads where control plane communication adds overhead

---

## References

- [What Is Amazon EKS?](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [EKS Best Practices](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [EKS Pricing](https://aws.amazon.com/eks/pricing/)
- [EKS Kubernetes Versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [EKS Anywhere](https://anywhere.eks.amazonaws.com/)
