# 2. Amazon EKS Core Architecture

## Overview

Amazon EKS separates responsibilities between AWS (control plane) and the customer (data plane). AWS operates the Kubernetes control plane across three Availability Zones in the customer's AWS account, while the customer manages worker nodes, pods, and application workloads.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AWS Region (e.g., us-east-1)                       │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │              AWS-Managed EKS Control Plane Account                      │    │
│  │                                                                         │    │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │    │
│  │  │    AZ-1a       │  │    AZ-1b       │  │    AZ-1c       │           │    │
│  │  │                │  │                │  │                │           │    │
│  │  │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │           │    │
│  │  │ │ API Server │ │  │ │ API Server │ │  │ │ API Server │ │           │    │
│  │  │ │ (kube-api)  │ │  │ │ (kube-api)  │ │  │ │ (kube-api)  │ │           │    │
│  │  │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │           │    │
│  │  │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │           │    │
│  │  │ │    etcd    │ │  │ │    etcd    │ │  │ │    etcd    │ │           │    │
│  │  │ │  (leader)  │ │  │ │(follower)  │ │  │ │(follower)  │ │           │    │
│  │  │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │           │    │
│  │  │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │           │    │
│  │  │ │ Scheduler  │ │  │ │ Scheduler  │ │  │ │ Scheduler  │ │           │    │
│  │  │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │           │    │
│  │  │ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │           │    │
│  │  │ │Ctrl Manager│ │  │ │Ctrl Manager│ │  │ │Ctrl Manager│ │           │    │
│  │  │ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │           │    │
│  │  └────────────────┘  └────────────────┘  └────────────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│           │                                                              │       │
│           │  EKS API Server Endpoint (Public/Private)                   │       │
│           │  (requester-managed ENIs in customer VPC)                   │       │
│           ▼                                                              │       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │       │
│  │                    Customer VPC (10.0.0.0/16)                       │  │       │
│  │                                                                     │  │       │
│  │  ┌──────────────────────┐  ┌──────────────────────────────────┐    │  │       │
│  │  │   Public Subnet      │  │        Private Subnet             │    │  │       │
│  │  │   (10.0.0.0/20)      │  │        (10.0.32.0/20)            │    │  │       │
│  │  │                      │  │                                    │    │  │       │
│  │  │  ┌────────────────┐  │  │  ┌────────────────────────────┐  │    │  │       │
│  │  │  │      ALB       │  │  │  │     Managed Node Group     │  │    │  │       │
│  │  │  └────────────────┘  │  │  │  ┌──────┐ ┌──────┐ ┌──────┐│  │    │  │       │
│  │  │  ┌────────────────┐  │  │  │  │Node 1│ │Node 2│ │Node 3││  │    │  │       │
│  │  │  │      NLB       │  │  │  │  │      │ │      │ │      ││  │    │  │       │
│  │  │  └────────────────┘  │  │  │  │┌────┐│ │┌────┐│ │┌────┐││  │    │  │       │
│  │  └──────────────────────┘  │  │  ││Pod ││ ││Pod ││ ││Pod │││  │    │  │       │
│  │                             │  │  │└────┘│ │└────┘│ │└────┘││  │    │  │       │
│  │                             │  │  │┌────┐│ │┌────┐│ │┌────┐││  │    │  │       │
│  │                             │  │  ││Pod ││ ││Pod ││ ││Pod │││  │    │  │       │
│  │                             │  │  │└────┘│ │└────┘│ │└────┘││  │    │  │       │
│  │                             │  │  │      │ │      │ │      ││  │    │  │       │
│  │                             │  │  └──────┘ └──────┘ └──────┘│  │    │  │       │
│  │                             │  │  ┌────────────────────────────┐  │    │  │       │
│  │                             │  │  │    Fargate Profile         │  │    │  │       │
│  │                             │  │  │  ┌──────┐ ┌──────┐        │  │    │  │       │
│  │                             │  │  │  │Pod   │ │Pod   │        │  │    │  │       │
│  │                             │  │  │  │(Fargate)│(Fargate)│    │  │    │  │       │
│  │                             │  │  │  └──────┘ └──────┘        │  │    │  │       │
│  │                             │  │  └────────────────────────────┘  │    │  │       │
│  │                             │  └──────────────────────────────────┘    │  │       │
│  │                             │                                          │  │       │
│  │  ┌──────────────────────────┼──────────────────────────────────────┐   │  │       │
│  │  │  AWS Services Integration                                     │   │  │       │
│  │  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐      │   │  │       │
│  │  │  │ ECR  │ │ EBS  │ │ EFS  │ │S3    │ │KMS   │ │Secrets│     │   │  │       │
│  │  │  │      │ │      │ │      │ │      │ │      │ │Mgr    │     │   │  │       │
│  │  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘      │   │  │       │
│  │  └───────────────────────────────────────────────────────────────┘   │  │       │
│  └─────────────────────────────────────────────────────────────────────┘  │       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Responsibility Table

| Component | Managed By | Purpose |
|-----------|-----------|---------|
| Kubernetes API Server | **AWS** | Accepts and processes all K8s API calls |
| etcd | **AWS** | Persistent backing store for cluster state |
| kube-scheduler | **AWS** | Assigns pods to nodes based on constraints |
| kube-controller-manager | **AWS** | Runs control loops, maintains desired state |
| Authentication | **AWS + Customer** | IAM authentication, RBAC authorization |
| Encryption at rest | **AWS** | KMS encryption for secrets and etcd |
| Certificate authority | **AWS** | Manages TLS certificates for API server |
| kubelet | **Customer** | Runs on each node, manages pods |
| kube-proxy | **Customer** | Maintains network rules for Services |
| Container Runtime | **Customer** | containerd (default) on each node |
| CoreDNS | **Customer** | Cluster DNS resolution |
| VPC CNI | **Customer** | Assigns VPC IP addresses to pods |
| AWS LB Controller | **Customer** | Manages ALB/NLB for Services/Ingress |
| EBS CSI Driver | **Customer** | Manages EBS volume lifecycle |
| EFS CSI Driver | **Customer** | Manages EFS volume lifecycle |

---

## Control Plane Components (AWS-Managed)

### Kubernetes API Server (kube-apiserver)

The API server is the front door to the Kubernetes control plane. Every `kubectl` command, every controller loop, and every kubelet heartbeat goes through the API server.

- **Runs in**: AWS-managed account, across 3 AZs
- **Endpoint**: Public, private, or both (customer-configurable)
- **Encryption**: All data encrypted at rest with KMS
- **Authentication**: IAM-based (STS tokens) mapped to Kubernetes identities
- **Rate limiting**: Built-in API priority and fairness

```bash
# Get the API server endpoint
aws eks describe-cluster --name my-cluster --query "cluster.endpoint"

# Test API server connectivity
kubectl cluster-info
```

### etcd

etcd is the consistent and highly available key-value store used as the backing store for all Kubernetes cluster data.

- **Runs in**: AWS-managed account, across 3 AZs
- **Replication**: 3 replicas (one per AZ)
- **Encryption**: Encrypted at rest using AWS KMS
- **Backup**: Automated by AWS
- **Access**: No direct customer access; accessed only through the API server

### Kubernetes Scheduler

The scheduler watches for unassigned pods and assigns them to nodes based on resource requirements, node affinity, taints/tolerations, and topology constraints.

- **Runs in**: AWS-managed account
- **Replication**: Active-passive across AZs
- **Customization**: Limited; customers can use scheduler profiles or custom schedulers

### Kubernetes Controller Manager

Runs control loops that watch the shared state of the cluster and make changes to move the current state toward the desired state. Examples include:

- **Deployment controller**: Manages ReplicaSets
- **Service controller**: Manages load balancers
- **Node controller**: Manages node lifecycle
- **Endpoint controller**: Manages Service endpoints
- **Namespace controller**: Manages namespace lifecycle

---

## Data Plane Components (Customer-Managed)

### Nodes

Nodes are EC2 instances (or Fargate tasks) that run the kubelet and container runtime. Each node joins the cluster by:

1. Running the bootstrap script (from AMI)
2. Registering with the API server
3. Receiving a kubeconfig for the kubelet
4. Starting kubelet and kube-proxy

### kubelet

The kubelet is the primary node agent that:

- Watches the API server for pods assigned to this node
- Creates and manages containers via the container runtime
- Reports node status (CPU, memory, disk, network)
- Executes liveness and readiness probes
- Manages pod volumes and secrets

### kube-proxy

kube-proxy maintains network rules on each node that implement Kubernetes Service abstraction:

- **iptables mode** (default): Uses iptables rules for DNAT and load balancing
- **IPVS mode**: Uses IPVS for better performance at scale (requires manual configuration)
- Manages ClusterIP, NodePort, and LoadBalancer rules

### Container Runtime

EKS uses **containerd** as the container runtime (CRI-compliant):

- Replaced Docker as the default in Kubernetes 1.24+
- Runs as a systemd service on each node
- Manages image pulling, container creation, and lifecycle
- Supports multi-arch images (amd64, arm64)

### CoreDNS

CoreDNS provides DNS resolution within the cluster:

- Resolves `<service-name>.<namespace>.svc.cluster.local`
- Runs as a Deployment with 2+ replicas
- Configurable via ConfigMap
- Can be scaled with cluster-proportional-autoscaler

---

## EKS Pod Identity / IAM Integration

### EKS Pod Identity (Recommended)

EKS Pod Identity is the newer, simpler mechanism for granting AWS permissions to pods:

```
Pod → ServiceAccount → Pod Identity Association → IAM Role → AWS Service
```

- AWS-managed agent runs on each node
- No OIDC provider configuration required
- Supports multi-tenancy (multiple roles per namespace)
- IAM role trust policy is simplified

### IRSA (IAM Roles for Service Accounts)

IRSA uses OIDC federation to grant AWS permissions:

```
Pod → ServiceAccount → OIDC Provider → IAM Role → AWS Service
```

- Requires an OIDC provider connected to the cluster
- Trust policy references the OIDC provider and service account
- More mature, wider region support
- Requires manual IAM role creation

---

## AWS Load Balancer Integration

| Type | Layer | Controller | Use Case |
|------|-------|-----------|----------|
| ALB | 7 | AWS LB Controller | HTTP/HTTPS ingress, path-based routing |
| NLB | 4 | AWS LB Controller | TCP/UDP, static IP, high performance |
| CLB | 4 | Legacy (deprecated) | Avoid; use NLB instead |

The AWS Load Balancer Controller watches for Kubernetes Service and Ingress resources and provisions AWS load balancers accordingly.

---

## Storage Integration

| Service | CSI Driver | Protocol | Use Case |
|---------|-----------|----------|----------|
| Amazon EBS | `ebs.csi.aws.com` | Block | Single-AZ, high IOPS |
| Amazon EFS | `efs.csi.aws.com` | File | Multi-AZ, shared access |
| Amazon FSx for Lustre | `fsx.csi.aws.com` | File | HPC, ML training |
| Mountpoint for S3 | `mountpoint-s3.csi.aws.com` | Object | S3-backed workloads |

---

## CloudWatch Integration

EKS integrates with CloudWatch for:

- **Control plane logging**: API, audit, authenticator, controller manager, scheduler logs
- **Container Insights**: CPU, memory, disk, network metrics for pods, nodes, and clusters
- **Prometheus metrics**: Via Amazon Managed Service for Prometheus
- **X-Ray tracing**: Distributed tracing via ADOT (AWS Distro for OpenTelemetry)

---

## VPC Networking

The VPC CNI plugin assigns real VPC IP addresses to pods:

- Each node has one primary ENI with primary and secondary private IPs
- Pods receive secondary IPs from the node's ENI
- Pods are directly routable within the VPC
- Security groups can be applied per-pod (via Security Groups for Pods)

```
VPC (10.0.0.0/16)
├── Public Subnet (10.0.0.0/20)
│   ├── ALB (10.0.0.10)
│   └── NAT Gateway (10.0.0.5)
└── Private Subnet (10.0.32.0/20)
    ├── Node 1 (10.0.32.10)
    │   ├── Pod A (10.0.32.11) - secondary IP
    │   └── Pod B (10.0.32.12) - secondary IP
    └── Node 2 (10.0.32.20)
        ├── Pod C (10.0.32.21) - secondary IP
        └── Pod D (10.0.32.22) - secondary IP
```

---

## Traffic Flow

### Internet → Pod

```
Internet → Route 53 → ALB/NLB → Target Group (IP targets) → Pod
```

### Pod → AWS Service (e.g., S3)

```
Pod → VPC CNI (VPC routing) → VPC Endpoint (Gateway/Interface) → S3
```

### Pod → Pod (Same Node)

```
Pod A → kube-proxy (iptables) → Pod B (loopback within node)
```

### Pod → Pod (Cross-Node)

```
Pod A → VPC CNI (secondary IP) → VPC routing → Node 2 → Pod C
```

### Pod → Internet

```
Pod → VPC routing → NAT Gateway → Internet Gateway → Internet
```

---

## Control Plane vs Data Plane

| Aspect | Control Plane | Data Plane |
|--------|--------------|------------|
| Managed by | AWS | Customer |
| Runs in | AWS-managed account | Customer VPC |
| Components | API server, etcd, scheduler, controller manager | Nodes, pods, kubelet, kube-proxy, CoreDNS |
| Networking | Requester-managed ENIs | VPC CNI (VPC-native) |
| Scaling | AWS-managed | Customer (managed node groups, Karpenter, Fargate) |
| Patching | AWS | Customer (node AMI, kubelet, add-ons) |
| Access | API endpoint only | SSH (if enabled), SSM, kubectl |
| Cost | $0.10/hr per cluster | EC2 instances, data transfer, storage |

---

## References

- [EKS Cluster Architecture](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html)
- [EKS Control Plane](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-resources.html)
- [VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [EKS Security](https://docs.aws.amazon.com/eks/latest/userguide/security.html)
- [EKS Networking](https://docs.aws.amazon.com/eks/latest/userguide/eks-compute-networking.html)
- [AWS Load Balancer Controller](https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html)
