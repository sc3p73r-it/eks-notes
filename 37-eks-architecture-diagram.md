# 37. EKS Architecture Diagram

## Overview

Diagram of the core Amazon EKS architecture: control plane, data plane (nodes), VPC networking, and application traffic flow.

---

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                             AMAZON EKS ARCHITECTURE (Simplified)                        │
│                                                                                         │
│  ┌───────────────────────┐     ┌───────────────────────┐                              │
│  │   Internet / Users    │────▶│  Route 53 / CloudFront │                              │
│  └───────────────────────┘     └───────────────────────┘                              │
│                                                                                         │
│  ┌───────────────────────┐                                                      │
│  │         VPC           │                                                      │
│  │  (10.0.0.0/16 private)│                                                      │
│  └────────────┬──────────┘                                                      │
│             │                                                               │
│             │  ┌─────────────────────────────────────────────────────┐     │
│             │  │   Public Subnets (3 AZs)                     │     │
│             │  │  ┌─────────────────┐   ┌─────────────────────┐ │     │
│             │  │  │  ALB / NLB      │   │  NAT Gateway        │ │     │
│             │  │  └─────────────────┘   └─────────────────────┘ │     │
│             │  └─────────────────────────────────────────────────────┘     │
│             │                                                               │
│  ┌────────────▼──────────┐                                                   │
│  │   EKS Control Plane   │                                                   │
│  │  (managed, 3 AZs)     │                                                   │
│  │  ┌─────────────────┐  │                                                   │
│  │  │ API Server      │  │                                                   │
│  │  │ etcd (key-value)│  │                                                   │
│  │  │ Scheduler       │  │                                                   │
│  │  │ Controller MGMT │  │                                                   │
│  │  └─────────────────┘  │                                                   │
│  └────────────┬──────────┘                                                   │
│             │                                                               │
│             │  ┌─────────────────────────────────────────────────────┐     │
│             │  │   Private Subnets (3 AZs)                     │     │
│             │  │  ┌─────────────────┐   ┌─────────────────────┐ │     │
│             │  │  │  EKS Nodes      │   │  EBS Volumes        │ │     │
│             │  │  │ (EC2 instances)│   │  (gp3, EFS, S3)     │ │     │
│             │  │  └─────────────────┘   └─────────────────────┘ │     │
│             │  └─────────────────────────────────────────────────────┘     │
│             │                                                               │
│  ┌────────────▼────────────────────────────────────────────────────▼────┐ │
│  │                   POD-TO-POD / SERVICE FLOW                            │ │
│  │                                                                       │ │
│  │  Pod (10.0.x.x) → Service VIP → ENI (on node) → Destination Pod      │ │
│  │                                                                       │ │
│  │  Networking: Amazon VPC CNI (assigns VPC IPs to pods)                │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                                         │
│  ┌───────────────────────┐                                                    │
│  │     Storage Options   │                                                    │
│  │  ┌─────────────────┐ │                                                    │
│  │  │  EBS (block)    │ │                                                    │
│  │  │  EFS (file, RWX)│ │                                                    │
│  │  │  S3 (object)    │ │                                                    │
│  │  └─────────────────┘ │                                                    │
│  └───────────────────────┘                                                    │
│                                                                                         │
│  ┌───────────────────────┐                                                    │
│  │    Add-ons / Services │                                                    │
│  │  • CoreDNS          │                                                    │
│  │  • kube-proxy       │                                                    │
│  │  • VPC CNI          │                                                    │
│  │  • EBS CSI Driver   │                                                    │
│  │  • EFS CSI Driver   │                                                    │
│  │  • LB Controller   │                                                    │
│  │  • Secrets Store CSI│                                                    │
│  └───────────────────────┘                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Key

| Layer | Purpose |
|-------|---------|
| **Control Plane** | AWS-managed (API server, etcd, scheduler, controller manager). Runs across 3 AZs. |
| **Data Plane** | EC2 instances (managed node groups, Fargate, or self-managed) running kubelet, kube-proxy, pods. |
| **VPC** | Customer-owned CIDR; pods get secondary ENI IPs from the VPC; no need for NAT for internal traffic. |
| **CNI** | Amazon VPC CNI; primary/secondary IP allocation; supports NetworkPolicy, prefix delegation. |
| **Load Balancing** | ALB (L7, HTTP/HTTPS, path-based routing, WAF) or NLB (L4, static IP, TCP/UDP passthrough). |
| **Storage** | EBS (block, single AZ), EFS (file, multi-AZ RWX), S3 (object). CSI drivers provision volumes. |
| **Add-ons** | CoreDNS, kube-proxy, VPC CNI, EBS/EFS CSI, AWS Load Balancer Controller, secrets-store-csi. |
| **IAM** | IRSA (IAM Roles for Service Accounts) for pod AWS permissions; EKS Access Entries for cluster access. |
| **Observability** | CloudWatch Container Insights, Prometheus + Grafana, Fluent Bit → CloudWatch/OpenSearch, X-Ray/ADOT. |

---

## Notes

- **Control plane** is AWS-managed; customer manages only the **data plane (nodes)**.
- **Public endpoint** optional; recommended to use **private endpoint** + bastion/SSM for node access.
- **Cross-AZ** resilience: nodes and load balancers spread across all AZs; control plane inherently multi-AZ.
- **NetworkPolicy** default-deny recommended; CNI must support it (VPC CNI does).
- **Pod-to-pod** traffic uses ENI secondary IPs; nodes act as network gateways.

---

## References

- [EKS Architecture](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/networking.html)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Amazon EKS Workshop — Essentials](https://www.eksworkshop.com/docs/fastpaths/)