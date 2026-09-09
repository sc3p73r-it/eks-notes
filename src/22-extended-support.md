# 22. EKS Extended Support

## Overview

Amazon EKS offers two support windows for Kubernetes versions: Standard Support and Extended Support. This section explains the support lifecycle, pricing, upgrade requirements, and decision factors.

---

## Support Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  Kubernetes Version Lifecycle                            │
│                                                                         │
│  Kubernetes version N released                                          │
│  │                                                                      │
│  ├── Standard Support (14 months)................................................................│
│  │                                                                      │
│  └── Extended Support (additional 12 months).........................│
│                                                                         │
│  After Extended Support ends → version is deprecated                    │
│  → delisted, cluster must be upgraded                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

| Phase | Duration | Example (K8s 1.28) |
|-------|----------|---------------------|
| Standard Support | 14 months | Sep 2023 → Nov 2024 |
| Extended Support | +12 months | Dec 2024 → Nov 2025 |
| Delisted | After Extended ends | Dec 2025+ |

---

## Standard Support

- **Duration**: 14 months from the version release date
- **Included**: Security patches, bug fixes, official AWS support
- **Cost**: Included in standard EKS pricing ($0.10/hr per cluster)

### What Happens on Standard Support End

- The version transitions to **Extended Support** automatically
- Pricing increases (see below)
- Security patches continue, but with additional cost

---

## Extended Support

- **Duration**: Additional 12 months
- **Included**: Security patches, bug fixes, official AWS support
- **Cost**: Additional per-cluster pricing

### Extended Support Pricing

| Region | Price per cluster (build hours) |
|--------|--------------------------------|
| us-east-1 (N. Virginia) | $0.06 per build hour |
| us-west-2 (Oregon) | $0.06 per build hour |
| Other regions | Varies slightly |

**Note**: Extended Support pricing is **additional** on top of standard EKS cluster pricing. Billing is per cluster per hour.

---

## Why Choose Extended Support?

| Scenario | Why Extended Support |
|----------|---------------------|
| Image/plugin incompatibility | Not ready to upgrade to next minor version |
| Vendor certification | Third-party cert lag for new K8s version |
| Testing cycle | In progress, need more time for validation |
| Regulatory constraint | Freeze of versions due to compliance |
| Seasonal demand | Avoid change during peak season |

---

## Why Return to Standard Support

| Scenario | Why Return |
|----------|------------|
| Cost reduction | Save the extended-support premium |
| Security | Patching quality better on supported versions |
| Ecosystem | Community tooling supports only new versions |
| Compliance | Security audit requires actively supported versions |
| Storage/compute | Node group AMIs and addons are validated for new versions |

---

## Upgrade Requirements

| Requirement | Action |
|-------------|--------|
| Must upgrade sequentially | Can't skip minor versions in place |
| Node groups must be within support window | Otherwise AWS may require replacement |
| Addons must be within support window | Otherwise features break |
| Extended Support requires opt-in | Pay if you transition automatically |

---

## Terraform Configuration

Terraform must define the cluster version and plan for upgrades:

```hcl
# Configure Terraform for a specific Kubernetes version
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.31"  # pin to a supported version

  vpc_config {
    subnet_ids = module.vpc.private_subnets
  }
}

# Node groups must match cluster major version
resource "aws_eks_node_group" "workers" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "workers"
  version         = "1.31"
  node_role_arn   = aws_iam_role.node_role.arn
  subnet_ids     = module.vpc.private_subnets

  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 1
  }
}
```

### Upgrade in Terraform

```bash
# Pin version
terraform apply -var='eks_version=1.32'

# Update node group version
terraform apply -var='node_group_version=1.32'
```

---

## Production Considerations

| Consideration | Recommendation |
|----------------|----------------|
| Version tracking | Document exact version + extended-support end date |
| Upgrade staging | Upgrade staging 4-6 weeks before production |
| Maintenance window | Use off-peak hours for control plane outage |
| Addon support | Verify addons support target version |
| Application compat | Test with Canary/Blue-Green deployments |
| Cost tracking | Track extended-support cost per cluster |
| Budget cycle | Plan annual Kubernetes upgrade cycle |

---

## Decision Checklist

**Choose Extended Support when:**
- Vendor certification pending for next minor version
- Testing not complete for workload compatibility
- Peak business period approaching
- Migration/unrelated critical project in flight

**Return to Standard Support when:**
- Next minor version tested and validated
- Ecosystem tooling (Helm charts, operators) supports target version
- Cost reduction is a priority
- Audit/compliance requires current security support

---

## AWS CLI / API Actions

```bash
# Check current version and support type
aws eks describe-cluster --name my-cluster \
  --query "cluster.kubernetesVersion"

# List supported versions, extended support status
aws eks describe-kubernetes-versions

# Update cluster version (prompts about extended support cost)
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.32
```

---

## References

- [EKS Version Lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [Kubernetes Versions Standard Support](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-standard.html)
- [Kubernetes Versions Extended Support](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-extended.html)
- [EKS Pricing](https://aws.amazon.com/eks/pricing/)