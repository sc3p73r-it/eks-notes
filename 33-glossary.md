# 33. Glossary

## Overview

Alphabetical glossary of important EKS, Kubernetes, and AWS terms.

---

## A

- **ACM (AWS Certificate Manager)** — AWS service for provisioning, managing, and renewing SSL/TLS certificates, commonly used to terminate TLS at ALB/CloudFront.
- **ALB (Application Load Balancer)** — AWS Layer 7 load balancer (HTTP/HTTPS) that routes traffic based on paths/hosts; integrated with EKS via the AWS Load Balancer Controller.
- **AMI (Amazon Machine Image)** — A template for launching EC2 instances; EKS provides EKS-optimized AMIs and Bottlerocket AMIs.
- **API Server (kube-apiserver)** — The Kubernetes control plane component that exposes the Kubernetes API; managed by AWS in EKS.
- **ARO (Amazon Resource Name)** — Unique Amazon Resource Identifier (e.g., `arn:aws:eks:us-east-1:123456789012:cluster/my-cluster`).
- **ASG (Auto Scaling Group)** — AWS service for automatic scaling of EC2 instances; used for EKS node groups.
- **Autoscaler** — Tools to scale pods (HPA) or nodes (Cluster Autoscaler / Karpenter).

## B

- **Blue/Green Deployment** — Deployment strategy where new version is deployed in parallel and traffic switched atomically.
- **Bottlerocket** — Open-source Linux distribution optimized for containers; supported as an EKS AMI.

## C

- **Canary Deployment** — Gradual rollout of a new version to a small percentage of traffic.
- **CIDR** — Classless Inter-Domain Routing notation for IP addresses/subnets (e.g., `10.0.0.0/16`).
- **CI/CD** — Continuous Integration/Continuous Deployment; automation for building, testing, and deploying.
- **Cluster Autoscaler** — Kubernetes component that adds/removes nodes (via AWS ASG) based on pending pods.
- **CNI (Container Network Interface)** — Standard for Kubernetes networking plugins; EKS uses Amazon VPC CNI by default.
- **ConfigMap** — Kubernetes resource for non-confidential configuration data.
- **Container Insights** — AWS CloudWatch feature that collects, aggregates, and summarizes metrics and logs from containers.
- **Control Plane** — Kubernetes control plane (API server, etcd, scheduler, controller manager); AWS-managed in EKS.
- **CoreDNS** — Cluster DNS resolver for Kubernetes (default DNS provider).
- **CPU Manager** — Kubernetes feature for CPU pinning.
- **CSI (Container Storage Interface)** — Standard for storage drivers; used by EBS/EFS/FSx CSI drivers.
- **Customer Managed Key (CMK)** — Customer-owned KMS key used for encryption.

## D

- **DaemonSet** — Kubernetes workload that runs a pod on every node.
- **Data Plane** — The worker nodes, pods, kubelet, CNI, and related runtime in Kubernetes.
- **Deployment** — Kubernetes controller providing declarative updates for pods (ReplicaSets).
- **Desired State** — The declared target state of infrastructure (compare to Current State); Kubernetes reconciles both.
- **Drain** — Kubernetes operation to evict pods from a node gracefully.

## E

- **EBS (Amazon Elastic Block Store)** — Block storage for EC2; main block storage option for EKS stateful applications.
- **EBS CSI Driver** — Container Storage Interface driver for EBS volumes on EKS.
- **EC2 (Amazon Elastic Compute Cloud)** — AWS virtual servers used for EKS worker nodes.
- **ECR (Amazon Elastic Container Registry)** — AWS container image registry, deeply integrated with EKS.
- **EFS (Amazon Elastic File System)** — AWS file storage (NFS) supporting ReadWriteMany for shared access.
- **EKS (Amazon Elastic Kubernetes Service)** — AWS-managed Kubernetes service.
- **EKS Access Entry** — Modern mechanism to grant IAM principals access to an EKS cluster.
- **EKS Anywhere** — AWS-managed Kubernetes for on-premises environments.
- **EKS Auto Mode** — EKS feature that automatically manages compute, storage, and networking.
- **eksctl** — CLI tool for creating and managing EKS clusters.
- **ENI (Elastic Network Interface)** — Virtual network interface attached to EC2 instances; pods use ENI secondary IPs with VPC CNI.
- **etcd** — Distributed key-value store used as Kubernetes' backing store.

## F

- **Fargate** — AWS serverless compute engine for containers (EKS + ECS).
- **Fargate Profile** — Defines namespace/label selectors for pods to run on Fargate.
- **Fluent Bit** — Lightweight log processor used as a DaemonSet to collect logs to CloudWatch/OpenSearch.

## G

- **Gateway API** — Modern, role-oriented Kubernetes API for service networking (replaces Ingress for some use cases).
- **GitOps** — Pattern where Git is the single source of truth and changes are applied via a controller (Argo CD, Flux).
- **Grafana** — Open-source metrics visualization/dashboards.
- **GuardDuty** — AWS threat detection service, including EKS control plane and runtime monitoring.
- **GWLB (Gateway Load Balancer)** — Layer 3/4 gateway load balancer; rarely used directly with EKS.

## H

- **Helm** — Kubernetes package manager (charts).
- **HPA (HorizontalPodAutoscaler)** — Kubernetes autoscaling for replicas based on metrics.
- **Hybrid Nodes** — EKS feature to add on-premises nodes to an AWS EKS cluster.

## I

- **IAM (Identity and Access Management)** — AWS identity service for users, roles, policies.
- **IAM Role** — AWS identity assumed by trusted principals; used for EKS cluster/node/pod roles.
- **Ingress** — Kubernetes resource for HTTP(S) routing to Services (often backed by ALB via controller).
- **Insights** — Kubernetes resource for HTTP(S) routing.
- **IRSA (IAM Roles for Service Accounts)** — TEAM technique to grant pods AWS permissions via OIDC federation.

## J

- **Job** — Kubernetes workload that runs to completion.
- **JSON** — Format used for structured logging and API output.

## K

- **Karpenter** — Open-source node provisioning tool from AWS; provisions EC2 nodes dynamically for pods.
- **Kubeconfig** — Kubernetes configuration file containing cluster/credential context.
- **kubelet** — Node agent managing pods and containers on each node.
- **kube-proxy** — Node component maintaining network rules for Kubernetes Services.
- **Kubernetes** — Open-source container orchestration platform.
- **Kustomize** — Native Kubernetes configuration management tool (overlays).
- **KMS (AWS Key Management Service)** — AWS service for managing encryption keys.

## L

- **Liveness Probe** — Kubernetes probe that restarts a container when it fails.
- **Local Volume** — EBS volumes attached to a specific node/AZ.

## M

- **Managed Node Group** — AWS-managed auto-scaling EC2 nodes for EKS.
- **Metrics Server** — Kubernetes API extension providing CPU/memory metrics (via resource metrics API).

## N

- **Namespace** — Kubernetes isolation resource used for multi-tenancy.
- **NetworkPolicy** — Kubernetes resource restricting pod-to-pod traffic.
- **NLB (Network Load Balancer)** — AWS Layer 4 (TCP/UDP) load balancer; supports static IPs.
- **Node** — Worker machine (EC2 instance) running kubelet and pods.
- **Node Group** — A group of EC2 nodes managed by EKS/ASG.

## O

- **OIDC (OpenID Connect)** — Authentication protocol; EKS uses an OIDC provider for IRSA.
- **OpenSearch** — Popular log/analytics engine for log aggregation.
- **OpenTelemetry (OTel)** — Open-source observability framework for metrics, logs, and traces.
- **Operator** — Kubernetes program/controller that automates management of an application/service.

## P

- **Parameter Store** — AWS Systems Manager service for configuration/secret parameters.
- **PDB (PodDisruptionBudget)** — Guarantees minimum available pods during voluntary disruptions.
- **PersistentVolume/PersistentVolumeClaim** — Kubernetes storage abstraction layer.
- **Pod** — Smallest deployable unit in Kubernetes (one or more containers).
- **Pod Security Standards (PSS)** — Privileged, Baseline, Restricted pod security policies.
- **Pod Security Admission** — Kubernetes admission controller enforcing Pod Security Standards.
- **Prometheus** — Time-series metrics collection/alerting system.

## R

- **RBAC (Role-Based Access Control)** — Kubernetes authorization model (Roles, RoleBindings, ClusterRoles, ClusterRoleBindings).
- **Readiness Probe** — Prevents traffic from reaching a pod until it is ready.
- **ReplicaSet** — Kubernetes controller maintaining a stable set of replica pods.
- **RTO/RPO** — Recovery Time Objective / Recovery Point Objective (DR metrics).

## S

- **S3 (Amazon Simple Storage Service)** — Durable object storage; used for logs, backups, static assets.
- **Secret** — Kubernetes object for sensitive data (base64-encoded).
- **Security Group** — AWS virtual firewall for EC2/ENI-level network filtering.
- **Self-Managed Node** — Customer-managed EC2 node joining an EKS cluster.
- **Service** — Kubernetes abstraction providing a stable virtual IP/DNS to pods.
- **ServiceAccount** — Kubernetes identity for pods (used with IRSA/Pod Identity).
- **Service Mesh** — Infrastructure layer for mTLS, observability, routing (Istio, Linkerd, Cilium).
- **Spot Instance** — Discounted EC2 instance that can be interrupted, useful for stateless workloads.
- **StatefulSet** — Kubernetes workload for stateful apps with stable identity/storage.
- **StorageClass** — Defines storage class/provisioner parameters (e.g., gp3, EFS).
- **STS (AWS Security Token Service)** — Issues temporary credentials (used in EKS auth).

## T

- **Taint** — Node setting that repels pods without matching tolerations.
- **Terraform** — HashiCorp Infrastructure as Code tool, commonly used to provision EKS.
- **TLS** — Transport Layer Security for encrypted communications.

## V

- **VPA (VerticalPodAutoscaler)** — Kubernetes autoscaling to adjust resource requests/limits.
- **VPC (Amazon Virtual Private Cloud)** — AWS virtual network used by EKS.
- **VPC CNI** — Amazon VPC Container Network Interface; assigns VPC IPs to pods.
- **VPC Endpoint** — Private connection to AWS services without NAT/Internet gateway (Gateway or Interface types).

## W

- **WAF (AWS WAF)** — Web application firewall (L7) for ALB/CloudFront.
- **WaitForFirstConsumer** — Storage provisioning mode where PV is created when a pod first uses the PVC.

## X

- **X-Ray** — AWS distributed tracing service for application traces.

## Z

- **Zone (AZ)** — Availability Zone; EKS control plane spans 3 AZs.
- **Zone-Redundant / Multi-AZ** — Architecture used across multiple Availability Zones for availability.

---

## References

- [GKE docs](https://kubernetes.io/docs/concepts/overview/components/)
- [AWS EKS terminology](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)