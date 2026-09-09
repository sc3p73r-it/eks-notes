# 28. EKS Production Best Practices

## Overview

A production-ready EKS cluster requires deliberate choices in security, reliability, performance, operations, and cost. This section is a comprehensive best-practices checklist based on AWS guidance and real-world experience.

---

## Security

### Least Privilege

- Use role-based access control (IAM roles + EKS access entries)
- Use namespace-scoped Roles/RoleBindings, avoid ClusterRole/admin by default
- Enable IRSA or Pod Identity for every workload with AWS access
- Rotate credentials regularly; use short-lived credentials in CI

### IAM

- Use AWS-managed SCPs or permission boundaries
- Require MFA for console and CLI users
- Use EKS Pod Identity for new deployments (simpler)
- Audit IAM with Access Analyzer

### RBAC

- Define Roles per team/namespace
- Enable audit logging
- Avoid granting `cluster-admin` to non-platform teams
- Use temporary tokens, avoid static kubeconfig in CI

### NetworkPolicy

- Adopt default-deny per namespace
- Allow only required pod-to-pod and egress traffic
- Use Calico or Cilium (or VPC CNI network policy) to enforce

### Encryption

- Enable KMS encryption on EKS secrets
- Enable EBS encryption by default
- Use EFS encryption
- Use ACM for TLS
- Encrypt Kubernetes Secrets at rest (envelope encryption)

### Secrets

- Centralize in AWS Secrets Manager / SSM
- Sync via External Secrets Operator
- Never store secrets in Git; use secret scanners in CI
- Rotate secrets regularly

### Image Scanning

- Enable ECR scanOnPush
- Use Amazon Inspector (enhanced scanning)
- Use Trivy/Snyk in CI on every build
- Gate deployment on critical/high vulnerabilities

### Pod Security

- Apply Pod Security Standards (Restricted for production)
- Run as non-root, drop capabilities, read-only root filesystem
- Set CPU/memory requests and limits
- Disable host network/pid sharing by default

---

## Reliability

### Multi-AZ

- Deploy nodes across 3 AZs
- Use multi-AZ subnets for LB targets
- Use multi-AZ NAT gateways

### PDB

- Define PodDisruptionBudgets for all critical workloads
- Use `minAvailable` for high-availability, `maxUnavailable` for skippable

### HPA

- Configure HPA on all burstable workloads
- Set CPU and memory targets
- Use custom metrics (Prometheus/SQS) where relevant

### Node Redundancy

- Minimum 3 nodes per node group
- Use managed node groups or Karpenter for auto replacement
- Keep min size ≥ 2 for critical node groups

### Backup

- Backup Kubernetes objects with Velero
- Enable EBS snapshots for stateful workloads
- Enable AWS Backup for EFS/RDS
- Back up to a different region for DR

### DR

- Define RTO/RPO targets
- Maintain a DR cluster (pilot light or warm standby)
- Test DR failover quarterly

---

## Performance

### Resource Requests/Limits

- Always set CPU/memory requests
- Set limits (avoid runaway memory)
- Use VPA recommendations to right-size
- Monitor with `kubectl top`

### Autoscaling

- Use Karpenter for dynamic workloads
- Use Cluster Autoscaler for node groups
- Enable HPA for pod-level scaling

### Node Sizing

- Choose instance types appropriate (compute vs memory vs GPU)
- Use ARM (Graviton) where compatible (~40% cost savings)
- Right-size to avoid idle nodes

### Networking

- Plan CIDR for pod IP scaling (prefix delegation if needed)
- Use VPC endpoints to reduce NAT traffic
- Use CLB→NLB/ALB for modern load balancing
- Enable cross-zone load balancing

### Storage

- Use gp3 volumes (cheaper, configurable)
- Use WaitForFirstConsumer for multi-AZ
- Use EFS for shared/multi-AZ reads

---

## Operations

### Monitoring

- Enable Container Insights
- Deploy Prometheus (kube-prometheus-stack)
- Monitor API server latency, node resources, pod health

### Logging

- Enable control plane logging (api, audit)
- Deploy Fluent Bit for container logs
- Set log retention to match compliance

### Alerting

- Alert on node NotReady, pod CrashLoop, API latency
- Route alerts to on-call (PagerDuty/Opsgenie)
- Test alert paths with chaos drills

### Upgrades

- Plan upgrades 4 weeks ahead
- Upgrade staging before production
- Track Kubernetes version + addon versions
- Use extended support only when necessary

### Documentation

- Maintain runbooks for common incidents
- Document architecture in a central knowledge base
- Keep change logs and maintenance windows

### Runbooks

- Node replacement
- Pod unschedulable
- API server connectivity
- DNS resolution issues
- Secret rotation
- AZ outage

---

## Cost

### Right Sizing

- Analyze utilization with `kubectl top` and CloudWatch
- Downsize or consolidate underutilized pods/nodes
- Use VPA recommendations

### Spot

- Deploy stateless, fault-tolerant workloads on Spot
- Use interruption handling (AWS Node Termination Handler)
- Start with 30% spot, iterate

### Savings Plans

- Purchase Compute Savings Plan for steady-state pods
- Purchase EC2 Instance Savings Plan for specific instances/NVMe
- Reserve GPU instances where predictable

### Reserved Capacity

- Use AWS Savings Plans for base capacity
- Combine spot + on-demand + savings plan for flexibility

### NAT Optimization

- Use VPC endpoints (Interface) for services accessed frequently
- Use Gateway endpoints for S3/DynamoDB (free)
- Consider consolidating NAT gateways for non-critical workloads

### Storage Optimization

- Use snapshots + tiering (S3 lifecycle)
- Use EFS IA for cold files
- Set CSI volume reclaim policies appropriately (Retain vs Delete)

---

## Post-Deployment Validation

```bash
# Validate cluster health
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl top nodes && kubectl top pods

# Emergency Profile
kubectl get events --sort-by='.metadata.creationTimestamp' -A

# Network validation
kubectl run test --image=curlimages/curl --rm -it \
  -- curl http://kubernetes.default.svc.cluster.local

# Security validation (RBAC can-i)
kubectl auth can-i --list -n <namespace>
```

---

## Reference Checklist

See [34-production-readiness-checklist.md](34-production-readiness-checklist.md) for the final production readiness checklist.

---

## References

- [EKS Best Practices](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [AWS Well-Architected EKS](https://docs.aws.amazon.com/wellarchitected/latest/kubernetes-workloads/welcome.html)
- [Container Security Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)