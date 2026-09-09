# 7. EKS Storage Deep Dive

## Overview

EKS provides multiple storage options through Container Storage Interface (CSI) drivers. The choice depends on access mode, performance requirements, availability, and cost. AWS offers EBS (block), EFS (file), FSx for Lustre (high-performance file), and Mountpoint for S3 (object).

---

## Storage Decision Matrix

| Feature | EBS | EFS | FSx for Lustre | S3 |
|---------|-----|-----|----------------|-----|
| Type | Block | File | File | Object |
| Access mode | ReadWriteOnce | ReadWriteMany | ReadWriteMany | ReadWriteMany |
| Availability | Single AZ | Multi-AZ | Single AZ | Multi-Region |
| Performance | High IOPS | Burst | Very high IOPS | Unlimited |
| Encryption | Yes | Yes | Yes | Yes |
| Snapshots | Yes | Yes | Yes | Versioning |
| Cost model | Per GB + IOPS | Per GB + requests | Per GB + throughput | Per request |
| Use case | Databases, stateful | Shared storage | HML, ML training | Backups, static assets |

---

## Amazon EBS

### EBS Volume Types for EKS

| Type | IOPS | Throughput | Use Case |
|------|------|-----------|----------|
| gp3 | 3,000-16,000 | 125-1,000 MB/s | General purpose (recommended) |
| gp2 | Up to 16,000 | 250 MB/s | Legacy (avoid for new) |
| io2 | Up to 64,000 | 1,000 MB/s | Mission-critical databases |
| io1 | Up to 64,000 | 1,000 MB/s | Legacy |
| st1 | Up to 500 | 500 MB/s | Throughput-intensive |
| sc1 | Up to 200 | 250 MB/s | Cold storage |

### StorageClass Examples

```yaml
# GP3 StorageClass (recommended)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Retain

# IO2 StorageClass (high performance)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Retain
```

### PVC Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-database-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi
```

### Volume Expansion

```yaml
# Enable expansion in StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-expandable
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
parameters:
  type: gp3
```

```bash
# Expand PVC
kubectl patch pvc my-database-pvc -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

### EBS Limitations

- **Single AZ**: EBS volumes are attached to instances in a single AZ
- **One attach**: Can only be attached to one instance at a time (except multi-attach with io1/io2)
- **Performance**: IOPS and throughput are provisioned, not unlimited
- **AZ migration**: Must be in same AZ as the node

---

## Amazon EFS

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    EFS File System                            │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                    AZ-1a                                │   │
│  │  ┌─────────────────────────────────────────────────┐  │   │
│  │  │  Mount Target (10.0.32.x)                        │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                    AZ-1b                                │   │
│  │  ┌─────────────────────────────────────────────────┐  │   │
│  │  │  Mount Target (10.0.48.x)                        │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                    AZ-1c                                │   │
│  │  ┌─────────────────────────────────────────────────┐  │   │
│  │  │  Mount Target (10.0.64.x)                        │  │   │
│  │  └─────────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-1234567890abcdef0
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
```

### EFS Performance Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| General Purpose | Default, low latency | Most workloads |
| Max I/O | Higher throughput, higher latency | Parallel processing |

### EFS Throughput Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| Bursting | Throughput scales with size | General workloads |
| Provisioned | Fixed throughput | Latency-sensitive |
| Elastic | Auto-scales with demand | Variable workloads |

### EFS Use Cases

- Shared configuration files across pods
- CMS media storage
- Development environments
- Log aggregation
- Machine learning training data

---

## Mountpoint for Amazon S3

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    S3 CSI Driver                              │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  S3 CSI Controller (Deployment)                        │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  S3 CSI Node (DaemonSet)                               │   │
│  │  Uses Mountpoint for Amazon S3                         │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: s3-existing-bucket
provisioner: s3.csi.aws.com
parameters:
  bucketName: my-bucket
  region: us-east-1
  authenticationSource: pod-identity
```

### Limitations

- Read-only by default
- No random write
- No renaming files
- No directory listing (cached)
- Not suitable for databases

---

## Backup Strategies

### EBS Snapshots

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot
driver: ebs.csi.aws.com
deletionPolicy: Retain
parameters:
  encrypted: "true"
---
# VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot
  source:
    persistentVolumeClaimName: my-database-pvc
```

### AWS Backup

```bash
# Create backup plan for EKS volumes
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "EKS-Volume-Backup",
  "Rules": [{
    "RuleName": "DailyBackup",
    "TargetBackupVaultName": "Default",
    "ScheduleExpression": "cron(0 12 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 180
  }]
}'
```

---

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| Storage class | Use `gp3` as default, `io2` for databases |
| Binding mode | Use `WaitForFirstConsumer` for multi-AZ |
| Encryption | Enable encryption at rest for all volumes |
| Reclaim policy | Use `Retain` for production data |
| Expansion | Enable `allowVolumeExpansion: true` |
| Snapshots | Schedule regular snapshots for stateful workloads |
| EFS | Use for shared storage, multi-AZ requirements |
| Monitoring | Monitor PVC usage, set alerts for capacity |

---

## Troubleshooting

```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc <pvc-name>

# Check PV status
kubectl get pv
kubectl describe pv <pv-name>

# Check storage class
kubectl get sc

# Check CSI driver pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver

# Check EBS volumes in AWS
aws ec2 describe-volumes --filters Name=tag:eks:pv-name,Values=<pv-name>

# Check volume attachment
aws ec2 describe-volume-attachments --volume-ids vol-0123456789abcdef0

# Check events
kubectl describe pod <pod-name> | grep -A5 Events
```

---

## References

- [EKS Storage](https://docs.aws.amazon.com/eks/latest/userguide/storage.html)
- [EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
- [EFS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)
- [Mountpoint for S3](https://docs.aws.amazon.com/eks/latest/userguide/s3-csi.html)
