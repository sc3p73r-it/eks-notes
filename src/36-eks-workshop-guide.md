# 36. EKS Workshop Guide

## Overview

The [Amazon EKS Workshop](https://www.eksworkshop.com/) is the official hands-on lab environment maintained by AWS (`aws-samples/eks-workshop-v2`). It provides a pre-configured AWS environment (an EKS cluster, an IDE, and a retail sample application called **EKS Workshop**) so you can run labs without building infrastructure yourself.

The workshop is organized into **two learning paths**:

| Path | Audience | Description |
|------|----------|-------------|
| **Amazon EKS Essentials** (Fast Paths) | Role-based, quick | Streamlined, role-based labs (Developer, Operator, Capability) powered by **EKS Auto Mode**. Minimal infrastructure to manage. |
| **Amazon EKS — Modular** | Comprehensive | Eight chapters (Intro through AI/ML) that explore every major EKS feature and AWS integration. |

---

## Learning Path 1: Amazon EKS Essentials (Fast Paths)

**Powered by EKS Auto Mode.** Amazon EKS Auto Mode extends AWS management beyond the control plane to also manage compute autoscaling, networking, load balancing, DNS, and block storage — so labs focus on learning rather than infrastructure setup.

### Prerequisites / Setup

1. **Setup** — either at an AWS event (pre-provisioned) or in your own account; the guide provides an IDE + cluster ready to use.
2. **Navigating the labs** — learn the lab conventions.
3. **Getting started** — deploy the retail sample application and explore `kubectl`.

### Developer Essentials

Targets developers deploying workloads. Labs:

| Lab | What you learn |
|-----|----------------|
| Exposing workloads with Ingress | Load balancers + Ingress to expose apps (ALB via LB controller) |
| Adding workload storage with EBS | Persistent storage via EBS CSI, StorageClasses, PVCs |
| Accessing AWS APIs securely from workloads | **EKS Pod Identity** (e.g., a pod reading/writing DynamoDB) |
| Autoscaling applications | **KEDA** (Kubernetes Event-Driven Autoscaling) |
| Accessing workload logs | Fluent Bit → CloudWatch; verify logs in CloudWatch |

### Operator Essentials

Targets cluster operators. Labs:

| Lab | What you learn |
|-----|----------------|
| Autoscaling with EKS Auto Mode | **Karpenter**-based cluster autoscaling in Auto Mode |
| Enabling secure Pod-to-Pod communication | **Network Policies** |
| Managing secrets with AWS Secrets Manager | Secrets Manager + External Secrets |
| Accessing AWS APIs securely from workloads | EKS Pod Identity |

### Capability Essentials

Targets platform engineers and DevOps. **Builds on Amazon EKS Capabilities** — a newer AWS offering (announced Nov 2025) where common platform components (ACK, Argo CD, kro) run as **fully managed control-plane components on AWS-owned infrastructure**. No Helm installs, no controller Deployments, no pod-level IRSA for the controllers — AWS handles scaling, patching, and upgrades.

| Capability | Lab scenario |
|------------|--------------|
| **ACK** (AWS Controllers for Kubernetes) | Provision a real DynamoDB table by applying a `Table` resource; migrate a microservice from in-cluster mock to AWS table via EKS Pod Identity |
| **Argo CD** | GitOps delivery from a pre-seeded AWS CodeCommit repo; sign in to the managed Argo CD UI via **AWS IAM Identity Center** |
| **kro** (Kube Resource Orchestrator) | Compose multiple apply steps into a single `CartsStack` via `ResourceGraphDefinition`; watch kro reconcile the whole graph |

---

## Learning Path 2: Amazon EKS — Modular

### Introduction

- **Setup** — provision the EKS cluster and IDE
- **Kubernetes Basics** — PODs, Deployments, ConfigMaps, Services
- **Kustomize** — overlay-based config management
- **Helm** — package/chart management

### Fundamentals

| Module | Labs |
|--------|------|
| Exposing applications | AWS Load Balancer Controller, Load Balancers, Ingress, **Gateway API** |
| Storage | **Amazon EBS**, **Amazon EFS**, **Mountpoint for Amazon S3**, **FSx for OpenZFS**, **FSx for Lustre**, **FSx for NetApp ONTAP** |
| Compute | Managed Node Groups (basics, **Cluster Autoscaler**, **Graviton/ARM**, **Spot instances**), **Karpenter**, **Fargate** |
| Workload Autoscaling | **HPA**, **Cluster Proportional Autoscaler**, **KEDA** |
| EKS Cluster Upgrades | Explore cluster upgrade process |

### Observability

| Module | Notes |
|--------|-------|
| View EKS console | Resource view in console |
| Logging | Control plane logs (`api`, `audit`, `authenticator`, `controllerManager`, `scheduler`) + pod logs via Fluent Bit |
| Observability with OpenSearch | Fluent Bit → OpenSearch |
| EKS open source observability | Prometheus + Grafana (`kube-prometheus-stack`) |
| Container Insights | CloudWatch Container Insights on EKS |
| Cost visibility with Kubecost | Cost allocation per namespace/workload |
| Chaos Engineering with EKS | Fault injection / resilience testing |

Also points to the [AWS Observability Accelerator](https://aws-observability.github.io/terraform-aws-observability-accelerator/) (Terraform/CDK modules for CloudWatch, AMP, AMG, ADOT).

### Security

| Module | Notes |
|--------|-------|
| Cluster Access Management API | IAM roles via Access Entries |
| IAM Roles for Service Accounts | IRSA |
| Amazon EKS Pod Identity | Pod-level AWS permissions |
| Secrets Management | Kubernetes Secrets vs **Secrets Manager** vs **Sealed Secrets** |
| Amazon GuardDuty for EKS | Threat detection |
| Pod Security Standards | `privileged` / `baseline` / `restricted` |
| Policy management with **Kyverno** | Admission policy management |

Referenced: [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/).

### Networking

| Module | Notes |
|--------|-------|
| Amazon VPC CNI | CNI basics |
| Network Policies | Pod-to-pod traffic control with VPC CNI |
| Security Groups for Pods | Per-pod security groups |
| Custom Networking | Pods on separate CIDR from nodes |
| Prefix Delegation | Scale pod density / reduce IP exhaustion |
| EKS Hybrid Nodes | On-premises nodes joining the cluster |

### Automation

| Module | Notes |
|--------|-------|
| GitOps | **Flux**, **Argo CD** |
| Control Planes | **ACK** (AWS Controllers for Kubernetes), **Crossplane**, **kro** (Kube Resource Orchestrator) |
| Continuous Delivery | AWS CodePipeline |
| Platform Engineering on EKS | Explore module |

### AI/ML on EKS

| Module | Notes |
|--------|-------|
| Large Language Models with vLLM | Serving LLMs on GPU (chatbot lab) |
| Inference with AWS Inferentia | Inferentia/Trainium accelerator inference |
| Operating EKS with Kiro CLI | Natural-language EKS operations (Kiro) |
| AI on EKS | Explore module (training, distributed training, ML pipelines) |

### Troubleshooting Scenarios (Preview)

Hands-on labs mirroring real AWS support cases:

| Lab | Scenario |
|-----|----------|
| ALB Controller | Ingress/LB not provisioning |
| Worker Nodes | `NotReady` / node join failures |
| DNS Resolution | CoreDNS / service DNS issues |
| Pod Issues | CrashLoopBackOff / ImagePullBackOff / Pending |

---

## Lab Environment Model

- Each environment provides a pre-provisioned EKS cluster + Cloud9-style IDE with all tools (kubectl, eksctl, aws CLI, helm) installed.
- Labs use a `~$prepare-environment <module>` command to bootstrap the specific module's resources.
- The **retail sample application** (`catalog`, `carts`, `checkout`, `orders`, `assets`, `ui`) is used throughout; microservices are instrumented to simulate real errors during troubleshooting labs.
- The modular labs are pick-and-mix — choose only the modules relevant to your skill level/role.

---

## Mapping to This Documentation

| Workshop module | Related doc in this repo |
|-----------------|--------------------------|
| Introduction / K8s basics | 25-kubernetes-resources.md, 26-application-deployment-example.md |
| Exposure (LB/Ingress/Gateway API) | 09-ingress-application-traffic.md, 04-networking-deep-dive.md (Gateway API) |
| Storage (EBS/EFS/Mountpoint S3/FSx) | 07-storage-deep-dive.md |
| Compute (MNG/Karpenter/Fargate/Spot/Graviton) | 03-cluster-types-compute.md |
| Workloads autoscaling (HPA/CPA/KEDA) | 08-scaling.md |
| Cluster upgrades | 21-upgrades.md, 22-extended-support.md |
| Observability (logging/OpenSearch/metrics/Cost/Chaos) | 10-observability.md, 11-monitoring-alerting.md, 12-logging-architecture.md, 13-control-plane-logging.md |
| Security (access/IRSA/Pod Identity/Secrets/GuardDuty/PSS/Kyverno) | 05-iam-security.md, 17-secrets-management.md, 16-security-scanning-devsecops.md |
| Networking (CNI/NetworkPolicy/SG-for-Pods/Custom/Prefix) | 04-networking-deep-dive.md |
| Automation (GitOps/ACK/Crossplane/kro/CodePipeline) | 14-deployment-cicd.md |
| AI/ML on EKS (vLLM/Inferentia/Kiro) | 30-comparison-tables.md, 01-executive-overview.md |
| Troubleshooting labs | 27-troubleshooting.md |
| EKS Essentials (Auto Mode paths) | 03-cluster-types-compute.md (Auto Mode), 30-comparison-tables.md |

---

## New Capabilities Flagged by the Workshop

Several tools in the workshop are newer and worth tracking in this repo:

- **Amazon EKS Capabilities** — managed control-plane components (ACK, Argo CD, kro) that run on AWS-owned infrastructure; removes Helm installs and controller maintenance from customer responsibility.
- **kro (Kube Resource Orchestrator)** — a Kubernetes control plane that composes resources via `ResourceGraphDefinition`.
- **KEDA (Kubernetes Event-Driven Autoscaling)** — event-driven pod autoscaling (Kafka, SQS, Prometheus, Cron, etc.).
- **Kiro CLI** — natural language (LLM-driven) interface for EKS cluster operations.
- **Kyverno** — Kubernetes-native admission/policy engine (alternative/complement to OPA Gatekeeper).
- **Cluster Proportional Autoscaler** — scales replica counts in proportion to cluster size (e.g., CoreDNS with large clusters).
- **Kubecost** — Kubernetes-native cost monitoring/allocation.

---

## Getting Started

```bash
# 1. Open the workshop
open https://www.eksworkshop.com/

# 2. Choose a path
#    - New to EKS or short on time:  docs/fastpaths/ (Developer or Operator)
#    - Comprehensive learning:        docs/introduction → docs/aiml

# 3. From a shell with AWS credentials, prerequisites kick off per lab
~$prepare-environment fundamentals/storage/ebs
kubectl apply -f storage-class.yaml
```

Estimate: the modular Fundamentals → Networking core takes roughly 3–5 hours; the Essentials paths take 1–2 hours each.

---

## References

- [Amazon EKS Workshop](https://www.eksworkshop.com/)
- [EKS Workshop GitHub (aws-samples/eks-workshop-v2)](https://github.com/aws-samples/eks-workshop-v2)
- [Amazon EKS Capabilities announcement](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-eks-capabilities/)
- [kro (Kube Resource Orchestrator)](https://kro.run/)
- [AWS Observability Accelerator](https://aws-observability.github.io/terraform-aws-observability-accelerator/)
- [CDK AWS Observability Accelerator](https://aws-observability.github.io/cdk-aws-observability-accelerator/)
- [One Observability Workshop](https://catalog.workshops.aws/observability/en-US)
- [KEDA](https://keda.sh/)
- [Kyverno](https://kyverno.io/)
- [Kubecost](https://www.kubecost.com/)
- [Kiro (natural language EKS operations)](https://github.com/awslabs/kiro)