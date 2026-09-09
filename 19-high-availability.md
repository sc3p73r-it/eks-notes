# 19. EKS High Availability

## Overview

High availability (HA) for EKS requires redundancy at every layer: control plane, worker nodes, pods, load balancers, and supporting AWS services. EKS control plane is inherently HA across 3 AZs; the customer is responsible for data plane HA.

---

## HA Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Multi-AZ HA Architecture                             │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     AWS Region                                     │  │
│  │                                                                   │  │
│  │  ┌──────────────────────┐   ┌──────────────────────┐  ┌─────────┐ │  │
│  │  │      AZ-1a           │   │      AZ-1b           │  │  AZ-1c  │ │  │
│  │  │  ┌────────────────┐  │   │  ┌────────────────┐  │  │┌──────┐ │ │  │
│  │  │  │  Public Subnet │  │   │  │  Public Subnet │  │  ││ALB   │ │ │  │
│  │  │  │  ALB           │  │   │  │  ALB           │  │  ││(AZ)  │ │ │  │
│  │  │  └────────────────┘  │   │  └────────────────┘  │  │└──────┘ │ │  │
│  │  │  ┌────────────────┐  │   │  ┌────────────────┐  │  │┌──────┐ │ │  │
│  │  │  │  Node Group A  │  │   │  │  Node Group B  │  │  ││NodeG │ │ │  │
│  │  │  │  ┌────┐┌────┐ │  │   │  │  ┌────┐┌────┐ │  │  ││C     │ │ │  │
│  │  │  │  │Pod ││Pod │ │  │   │  │  │Pod ││Pod │ │  │  ││┌────┐│ │ │  │
│  │  │  │  └────┘└────┘ │  │   │  │  └────┘└────┘ │  │  │││Pod ││ │ │  │
│  │  │  └────────────────┘  │   │  └────────────────┘  │  ││└────┘│ │ │  │
│  │  └──────────────────────┘   └──────────────────────┘  │└──────┘ │ │  │
│  └────────────────────────────────────────────────────────┴─────────┘ │  │
│                                                                         │  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────────┐ │  │
│  │  NAT GW (AZ-1a) │  │  NAT GW (AZ-1b) │  │  VPC Endpoints (S3,    │ │  │
│  │                 │  │                 │  │  ECR, STS, Secrets Mgr)│ │  │
│  └─────────────────┘  └─────────────────┘  └────────────────────────┘ │  │
│                                                                         │  │
│  Control Plane: AWS-managed, HA across 3 AZs (API server + etcd)       │  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Multi-AZ Design

- **Subnets**: 3 public + 3 private subnets across 3 AZs
- **Node Groups**: Deploy across all 3 AZs
- **NAT Gateways**: One per AZ (no single point of failure)
- **Load Balancers**: ALB/NLB automatically distributes across AZs
- **Databases**: Multi-AZ RDS/Aurora

---

## Control Plane HA

EKS control plane is automatically HA:

| Component | HA Mechanism |
|-----------|--------------|
| API Server | Multiple replicas across 3 AZs |
| etcd | 3 replicas, Raft consensus |
| Scheduler | Multiple replicas, leader election |
| Controller Manager | Multiple replicas, leader election |
| Certificates | AWS-managed, auto-renewed |

---

## Worker Node HA

### Node Group Design

```bash
# Create multi-AZ node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --instance-types m5.large \
  --scaling-config minSize=6,maxSize=30,desiredSize=9 \
  --subnets subnet-a subnet-b subnet-c
```

### Minimum Node Configuration

- At least 3 nodes, one per AZ
- Minimum 2 nodes per AZ for critical workloads

---

## Pod Anti-Affinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 6
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values: [my-app]
            topologyKey: topology.kubernetes.io/zone
```

---

## Topology Spread Constraints

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 9
  template:
    metadata:
      labels:
        app: my-app
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
```

---

## PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

---

## Load Balancer HA

- ALB/NLB spans multiple AZs automatically
- Health checks route traffic away from unhealthy targets
- Use cross-zone load balancing for even distribution
- For NLBs, use one NLB per AZ + TargetGroup per zone for strict zonality

---

## Database HA

### RDS Multi-AZ

```bash
aws rds create-db-instance \
  --db-instance-identifier my-db \
  --multi-az \
  --engine postgres \
  --db-instance-class db.t3.medium \
  --allocated-storage 100
```

### Amazon Aurora

```bash
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora \
  --engine aurora-postgresql \
  --engine-version 16.1 \
  --availability-zones us-east-1a us-east-1b us-east-1c \
  --master-username admin \
  --master-user-password 'Password123!'
```

---

## Storage HA

- **EBS**: Single-AZ only, use snapshots for DR. For multi-AZ databases, use AWS-managed storage (RDS).
- **EFS**: Multi-AZ by nature (mount targets per AZ)
- **S3**: 11 nines durability, cross-region replication available

---

## NAT Gateway HA

```bash
# Create NAT Gateways in each AZ (recommended)
aws ec2 create-nat-gateway \
  --subnet-id subnet-a \
  --allocation-id eipalloc-a --tags Key=AZ,Value=1a

aws ec2 create-nat-gateway \
  --subnet-id subnet-b \
  --allocation-id eipalloc-b --tags Key=AZ,Value=1b

aws ec2 create-nat-gateway \
  --subnet-id subnet-c \
  --allocation-id eipalloc-c --tags Key=AZ,Value=1c
```

---

## Failure Scenario Analysis

| Failure | Detection | Recovery | RTO |
|---------|-----------|----------|-----|
| Pod failure | Liveness probe, restart | Kubelet restarts, Deployment controller | Seconds |
| Node failure | NotReady, cloud-controller-manager | Nod replacement by ASG, pending pods rescheduled | Minutes |
| AZ failure | ALB health check fails, node loss | Pods on remaining AZs, NLB failover | Minutes |
| Application failure | Readiness probe | Load balancer stops routing, Deployment rollout | Seconds-minutes |
| Load balancer failure | CloudWatch health | Recreate ALB, DNS update | Minutes |
| Database failure | RDS/App errors | Multi-AZ failover | Minutes |

### Node Failure Recovery Flow

1. Kubelet detects node failure (heartbeat timeout)
2. Cloud-controller-manager marks node NotReady
3. Pending pods are rescheduled to healthy nodes
4. EBS volumes remain attached (may require detach/reattach)
5. Autoscaler replaces the failed node

### AZ Failure Recovery Flow

1. Pods in failed AZ become unavailable
2. ALB/NLB health checks fail
3. Remaining healthy AZs absorb traffic
4. PodDisruptionBudgets may block new pod scheduling (review PDB minAvailable)
5. Rebalance via topology spread constraints

---

## RTO/RPO Targets

| Level | RTO | RPO |
|-------|-----|-----|
| Single pod | < 1 min | 0 (stateless) |
| Single node | < 5 min | 0 |
| Single AZ | < 15 min | Depends on storage |
| Region failover | < 1 hour | Depends on DR strategy |

---

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| Subnets | 3 AZs, public + private |
| Node groups | 3+ nodes, spread across AZs |
| Pod placement | Anti-affinity + topology spread |
| PDB | Always define, avoid blocking critical deploys |
| Load balancer | ALB/NLB across all AZs |
| Database | Multi-AZ RDS/Aurora |
| NAT | One NAT GW per AZ |
| Storage | Regular snapshots, cross-region replication for critical |
| Monitoring | Failover drills, chaos engineering |

---

## References

- [EKS Best Practices - Reliability](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [Multi-AZ](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html)
- [RDS Multi-AZ](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [PDB](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)