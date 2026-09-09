# 23. EKS Cost Optimization

## Overview

EKS cost optimization requires understanding the full cost model: cluster pricing, EC2 compute, storage, networking, and supporting services. Optimizing involves right-sizing, Spot adoption, savings plans, storage tiering, and data transfer reduction.

---

## Cost Model Breakdown

| Component | Cost Driver | Monthly Example (us-east-1) |
|-----------|-------------|------------------------------|
| EKS cluster | Per-cluster pricing | $73 (single cluster) |
| EC2 nodes | Instance type × count × hours | $600-2000 per cluster |
| Fargate | vCPU + memory + storage hours | $200-500 |
| EBS | GB-month + IOPS | $50-200 |
| EFS | GB-month + request cost | $20-100 |
| S3 | Storage + requests + egress | $10-100 |
| NAT Gateway | Per-GW-hour + data processed | $32 + $2.50/hr traffic |
| Load balancer | Per-LB-hour + LCU | $16-50 per LB |
| Data transfer | Per-GB (igress/egress) | Variable |
| CloudWatch | Metrics + logs + alerts | $20-100 |
| CloudTrail | First copy free, S3 storage | $5-20 |
| ECR | Storage + data transfer | $10-50 |
| KMS | $1/CMK + per-request | $3-10 |
| WAF | Per-rule + per-request | $5-20 |
| CloudFront | Data transfer + requests | $10-100 |

---

## Detailed Cost Breakdown

### EC2 Instance Pricing (us-east-1, On-Demand)

| Instance | vCPU | Memory (GB) | Monthly (730h) |
|----------|------|-------------|----------------|
| m5.large | 2 | 8 | $75 |
| m5.xlarge | 4 | 16 | $151 |
| m5.2xlarge | 8 | 32 | $302 |
| m6g.large (ARM) | 2 | 8 | $61 |
| m6g.xlarge (ARM) | 4 | 16 | $122 |
| c5.xlarge | 4 | 8 | $133 |
| r5.xlarge | 4 | 32 | $201 |
| t3.large | 2 | 8 | $53 |

### Spot Instance Savings

| Instance | On-Demand | Spot (approx) | Savings |
|----------|-----------|---------------|---------|
| m5.large | $75/mo | $22/mo | ~70% |
| m5.xlarge | $151/mo | $45/mo | ~70% |
| c5.xlarge | $133/mo | $40/mo | ~70% |
| r5.xlarge | $201/mo | $60/mo | ~70% |

### Savings Plans

| Plan | Commitment | Savings vs On-Demand |
|------|------------|---------------------|
| EC2 Instance Savings Plan | 1-yr | ~35% |
| EC2 Instance Savings Plan | 3-yr | ~40% |
| Compute Savings Plan | 1-yr | ~30% |
| Compute Savings Plan | 3-yr | ~35% |

---

## Example Monthly Cost Models

### Model A: Production Cluster (3-5 environments)

| Item | Quantity | Cost/Month |
|------|----------|------------|
| EKS control plane | 1 cluster | $73 |
| m5.xlarge nodes | 9 | $1,359 |
| EBS gp3 volumes | 500 GB | $40 |
| ALB | 2 | $32 |
| NAT Gateway | 3 | $96 |
| Data transfer | 1 TB | $90 |
| CloudWatch logs | 10 GB | $50 |
| CloudTrail | - | $10 |
| ECR | 10 GB | $1 |
| KMS | 2 keys | $2 |
| **Total** | | **~$1,753/mo** |

### Model B: Cost-Optimized Cluster

| Item | Quantity | Cost/Month |
|------|----------|------------|
| EKS control plane | 1 cluster | $73 |
| m5.large Spot nodes | 9 | $201 |
| EBS gp3 volumes | 200 GB | $16 |
| ALB | 1 | $16 |
| NAT Gateway | 1 | $32 |
| CloudWatch (reduced) | 3 GB | $20 |
| **Total** | | **~$358/mo** |

### Model C: Fargate-only (serverless)

| Item | Cost/Month |
|------|------------|
| EKS control plane | $73 |
| 3 pods, 1 vCPU/2GB, 24/7 | $90 |
| 10 pods, 1 vCPU/2GB, 10% load | $9 |
| EFS (for persistence) | $22 |
| **Total** | **~$194/mo** |

---

## Cost Optimization Strategies

### 1. Right-Sizing

- Use `kubectl top` and VPA recommendations to right-size
- Avoid over-provisioning requests
- Consolidate under-utilized nodes

### 2. Spot Instances

- Use Spot for stateless, fault-tolerant workloads
- Combine with interruption handler + rebalance
- Target ~30-50% of workload on Spot

### 3. Savings Plans

- Purchase Compute Savings Plan for predictable base capacity
- Reserve GPU instances (often costly)
- Use Savings Plans with account-level sharing

### 4. Storage Optimization

- Use gp3 (cheaper than gp2, more configurable)
- Use snapshots + tier/archive for cold storage (S3 Glacier)
- Use EFS Infrequent Access for cold files
- Enable EBS lifecycle rules, enforce `Retain` for critical only

### 5. NAT Optimization

- Replace NAT Gateways with VPC endpoints (Interface) for common services
- Use Gateway endpoints for S3/DynamoDB (free)
- Consolidate NAT Gateways in single AZ for non-critical workloads

### 6. Data Transfer

- Keep traffic within region (use INTERNAL ALB, VPC endpoints)
- Use CloudFront for edge caching to reduce egress
- Compress logs, archive old logs to S3
- Use cross-region data transfer strategically

### 7. Load Balancer Optimization

- Remove idle LBs, target groups for old services
- Use ALB for HTTP, NLB for TCP/static IP
- Use self-managed internal ALB endpoints where appropriate

### 8. CloudWatch Logs Cost Control

- Set retention (e.g., 30-90 days)
- Export old logs to S3 (cheap)
- Use Log Insights sparingly, enable contributor insights only when needed

### 9. Cluster Consolidation

- Merge multiple clusters into fewer, larger clusters
- Use namespaces + tenant isolation instead of cluster-per-team (when appropriate)
- Note trade-off: cluster blast radius and control

---

## Kubernetes Resource Management for Cost

### Resource Requests/Limits

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      resources:
        requests:
          cpu: 250m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 512Mi
```

**Apply these to reduce over-provisioning by ~30-50%** in test environments.

### Vertical Pod Autoscaler for Right-Sizing

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  updatePolicy:
    updateMode: "Auto"
```

---

## Cost Monitoring and Reporting

### AWS Cost Explorer

```bash
# List workloads by cluster
aws ce get-cost-and-usage \
  --time-period Start=2025-01-01,End=2025-01-31 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by '[{"Type":"TAG","Key":"eks:cluster-name"}]'
```

### EKS Cost Monitoring (OpenCost/Kubecost)

```bash
# Install Kubecost (open source)
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace
```

---

## Production Recommendations

| Priority | Action | Impact |
|----------|--------|--------|
| High | Enable Spot for stateless workloads | 60-90% compute savings |
| High | Right-size via VPA recommendations | 20-40% compute savings |
| High | Use gp3 storage with lifecycle rules | 20-50% storage savings |
| Medium | Use Savings Plans for base compute | 30-40% savings |
| Medium | Reduce NAT Gateway usage with VPC endpoints | $30-90/mo per endpoint |
| Medium | Set CloudWatch retention to 30-90 days | 50-70% log cost savings |
| Low | Merge clusters where appropriate | $73/cluster savings |
| Low | Consolidate load balancers | $16-50/LB savings |

---

## References

- [EKS Pricing](https://aws.amazon.com/eks/pricing/)
- [Cost Optimization on EKS](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [AWS Well-Architected - Cost](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [EC2 Spot best practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)