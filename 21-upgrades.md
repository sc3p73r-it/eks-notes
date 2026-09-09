# 21. EKS Upgrades

## Overview

Upgrading an EKS cluster involves sequential upgrades of the control plane, add-ons, node groups, and AMIs. AWS supports only specific Kubernetes versions, and any cluster outside the supported window cannot be upgraded—it must be recreated.

---

## Kubernetes Version Policy

| Policy | Value |
|--------|-------|
| Standard Support | 14 months from release |
| Extended Support | +12 months with additional pricing |
| Supported versions | Typically N-3 to N (latest available) |
| Skip versions | No, must upgrade one minor version at a time |

---

## Upgrade Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        EKS Upgrade Process                               │
│                                                                         │
│  1. Pre-Upgrade Checks                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Check current version vs target                              │    │
│  │  - Review API deprecations                                      │    │
│  │  - Review add-on compatibility                                  │    │
│  │  - Check application compatibility                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                          │                                              │
│  2. Backup                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Velero backup (config + data)                               │    │
│  │  - EBS snapshots                                                │    │
│  │  - Current Terraform state                                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                          │                                              │
│  3. Control Plane Upgrade                                               │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  aws eks update-cluster-version --name my-cluster               │    │
│  │  - AWS upgrades API server, etcd, scheduler, controller manager│    │
│  │  - Can take 30-60 minutes                                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                          │                                              │
│  4. Add-on Upgrades                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                          │                                              │
│  5. Node Group Upgrade                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  aws eks update-nodegroup-version --cluster-name my-cluster     │    │
│  │  - Rolling update with surge                                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                          │                                              │
│  6. Application Validation                                              │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  - Run smoke tests                                               │    │
│  │  - Check pod health                                              │    │
│  │  - Monitor metrics                                               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Pre-Upgrade Checks

### Check Current Versions

```bash
# Cluster version
aws eks describe-cluster --name my-cluster --query "cluster.version"

# Node group versions
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name production-nodes \
  --query "nodegroup.version"

# Addon versions
aws eks describe-addon --cluster-name my-cluster --addon-name vpc-cni
```

### Check API Deprecations

```bash
# Check for removed API versions (1.16 → 1.22 removed common APIs)
kubectl get --raw "/api" | jq '.versions'
kubectl api-resources --request-timeout=30s

# Use kubent (Kubernetes deprecated API checker)
kubent
```

### Review Application Compatibility

```bash
# Check pod security context requirements
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.securityContext}{"\n"}{end}'

# Check for use of removed API fields
kubectl get --raw /apis/apps/v1/deployments | jq '.items[].spec'
```

---

## 2. Backup

```bash
# Velero full backup
velero backup create pre-upgrade-backup --ttl 168h

# EBS snapshot (if stateful)
for vol in $(aws ec2 describe-volumes --filters "Name=tag:Name,Values=eks-*" --query 'Volumes[].VolumeId' --output text); do
  aws ec2 create-snapshot \
    --volume-id $vol \
    --description "Pre-upgrade backup"
done
```

---

## 3. Control Plane Upgrade

```bash
# Upgrade cluster to next version
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.31

# Check upgrade status
aws eks describe-cluster-version --name my-cluster

# Wait for upgrade to complete
kubectl get nodes --request-timeout=30s
```

### API Server Unavailability During Upgrade

During control plane upgrade, the API server may be briefly unavailable (~30-60 min). Plan maintenance windows accordingly.

---

## 4. Add-on Upgrades

```bash
# List compatible addon versions
aws eks describe-addon-versions \
  --kubernetes-version 1.31 \
  --addon-name vpc-cni

# Update addon to specific version
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE

# Update all addons
for addon in vpc-cni kube-proxy coredns; do
  aws eks update-addon \
    --cluster-name my-cluster \
    --addon-name $addon \
    --resolve-conflicts OVERWRITE
done
```

---

## 5. Node Group Upgrade

```bash
# Upgrade node group (updates AMI + kubelet)
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes \
  --force

# Or create a new node group and migrate
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes-v2 \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --instance-types m5.large \
  --scaling-config minSize=3,maxSize=10,desiredSize=3 \
  --subnets subnet-a subnet-b subnet-c

# Drain old node group
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

---

## 6. Application Validation

```bash
# Check pod health
kubectl get pods -A | grep -v Running

# Check resource usage
kubectl top nodes
kubectl top pods --sort-by=cpu

# Run integration tests
./run-smoke-tests.sh
```

---

## Upgrade Risks & Rollback

| Risk | Impact | Mitigation |
|------|--------|-----------|
| API deprecation | Application failure | Pre-upgrade API check |
| Manifest images require old API | Crash | Update manifests first |
| Addon incompatibility | CNI/DNS failure | Upgrade addons in staging first |
| Node version lag | Kubelet/API mismatch | Keep nodes within N-2 |
| Pod security | Scheduler rejects pods | Pre-upgrade pod security audit |
| Stateful app storage | Volume attachment issues | Back up volumes before upgrade |

### Rollback Considerations

- Kubernetes control plane upgrades are **not rollback-able** once started
- Node group upgrades can be rolled back by using the old AMI/launch template
- Always keep the previous Terraform/launch template version
- Restore from Velero backup if applications fail

---

## Upgrade Frequency

| Cluster | Upgrade Frequency | Maintenance Window |
|---------|-------------------|--------------------|
| Development | 1/month | Anytime |
| Staging | 1/quarter | Off-peak |
| Production | Every 12-18 months | 2 AM - 6 AM |

---

## Production Upgrade SOP

1. **Schedule** upgrade 4 weeks in advance, notify stakeholders
2. **Backup** cluster with Velero
3. **Review** upgrade notes, API deprecations
4. **Upgrade staging** first, validate 1 week
5. **Upgrade production** control plane (30-60 min)
6. **Upgrade add-ons** sequentially with validation between
7. **Upgrade node groups** with surge strategy
8. **Validate** all applications, monitor CloudWatch for 24h
9. **Document** version, AMI, addon versions in release notes

---

## References

- [EKS Version Lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [EKS Standard/Extended Support](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-standard.html)
- [Kubernetes Deprecation Guide](https://kubernetes.io/blog/2022/11/18/upcoming-changes-in-kubernetes-1-26/)