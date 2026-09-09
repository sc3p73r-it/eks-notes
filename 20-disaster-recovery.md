# 20. EKS Disaster Recovery

## Overview

Disaster recovery (DR) for EKS requires planning for both cluster and application recovery. A robust DR strategy combines backups (Velero, EBS snapshots, AWS Backup), infrastructure as code (Terraform), and GitOps for declarative cluster state.

---

## DR Strategies

| Strategy | RPO | RTO | Cost | Complexity |
|----------|-----|-----|------|-----------|
| Backup & Restore | Hours | Hours-days | Low | Low |
| Pilot Light | Minutes-hours | Hours | Medium | Medium |
| Warm Standby | Minutes | Minutes-hours | High | High |
| Multi-Site Active/Active | Minutes | Minutes | Very high | Very high |

---

## DR Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Multi-Region DR Architecture                        │
│                                                                         │
│  ┌───────────────────────────┐      ┌─────────────────────────────┐   │
│  │     Primary Region         │      │    DR Region                 │   │
│  │     (us-east-1)            │      │    (us-west-2)               │   │
│  │                            │      │                             │   │
│  │  ┌─────────────────────┐  │      │  ┌──────────────────────┐  │   │
│  │  │  ECR (images)       │  │  ──▶ │  │  ECR (replicated)     │  │   │
│  │  └─────────────────────┘  │      │  └──────────────────────┘  │   │
│  │  ┌─────────────────────┐  │      │  ┌──────────────────────┐  │   │
│  │  │  EKS Cluster        │  │      │  │  EKS Cluster (warm)   │  │   │
│  │  │  + GitOps (ArgoCD)  │  │      │  │  + GitOps (ArgoCD)    │  │   │
│  │  └─────────────────────┘  │      │  └──────────────────────┘  │   │
│  │  ┌─────────────────────┐  │      │  ┌──────────────────────┐  │   │
│  │  │  RDS (Primary)      │  │  ──▶ │  │  RDS (Replica→Promote)│  │   │
│  │  └─────────────────────┘  │      │  └──────────────────────┘  │   │
│  │  ┌─────────────────────┐  │      │  ┌──────────────────────┐  │   │
│  │  │  EBS Snapshots      │  │  ──▶ │  │  EBS Snapshots       │  │   │
│  │  │  (cross-region)     │  │      │  │  (restore on demand) │  │   │
│  │  └─────────────────────┘  │      │  └──────────────────────┘  │   │
│  │  ┌─────────────────────┐  │      │  ┌──────────────────────┐  │   │
│  │  │  S3 (Velero backup) │  │  ──▶ │  │  S3 (replicated)      │  │   │
│  │  └─────────────────────┘  │      │  └──────────────────────┘  │   │
│  │  ┌─────────────────────┐  │      │  ┌──────────────────────┐  │   │
│  │  │  Terraform State    │  │  ──▶ │  │  Terraform State      │  │   │
│  │  │  (S3 backend)       │  │      │  │  (shared/config)      │  │   │
│  │  └─────────────────────┘  │      │  └──────────────────────┘  │   │
│  └───────────────────────────┘      └─────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Velero Backup & Restore

### Architecture

Velero backs up Kubernetes resources (Deployments, Services, PVC references, ConfigMaps, Secrets) and optionally snapshots PVs via CSI or cloud provider plugins.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Velero Architecture                         │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │  Velero      │───▶│   S3 Bucket  │    │  AWS Plugins │     │
│  │  Server      │    │  (backups)   │    │  (snapshots) │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
│                                                                 │
│  K8s API → Backup resources (YAML) → S3                        │
│  EBS volumes → Snapshot via AWS plugin → EBS snapshot          │
└─────────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# Install Velero CLI
curl -L https://github.com/vmware-tanzu/velero/releases/latest/download/velero-v1.14.0-linux-amd64.tar.gz | tar -xzf -

# Install Velero server with AWS plugin
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket eks-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-volume-snapshots=true \
  --no-secret
```

### Create an IAM role for Velero

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:DeleteSnapshot",
        "ec2:DescribeSnapshots",
        "ec2:DescribeVolumes"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::eks-backups",
        "arn:aws:s3:::eks-backups/*"
      ]
    }
  ]
}
```

### Backing Up

```yaml
# Schedule backups
velero schedule create daily-backup \
  --schedule "0 2 * * *" \
  --include-namespaces production \
  --ttl 168h \
  --default-volumes-to-fs-backup=false

# One-time backup
velero backup create full-backup \
  --include-namespaces production \
  --ttl 168h
```

### Restoring

```bash
# List backups
velero backup get

# Describe backup
velero backup describe full-backup

# Restore
velero restore create --from-backup full-backup

# Verify restore
velero restore get
kubectl get pods -n production
```

---

## EBS Snapshots

### Automated Snapshot Lifecycle

```bash
# Create snapshot lifecycle rule for volumes
aws ec2 create-snapshots \
  --description "EKS volume" \
  --instance-specification InstanceId=i-0123456789abcdef0 \
  --tag-specifications '["ResourceType=snapshot","Tags=[{Key=Purpose,Value=EKS-Backup}]"]'
```

### Cross-Region Snapshot Copy

```bash
# Copy snapshot to DR region
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-0123456789abcdef0 \
  --source-description "EKS volume backup" \
  --description "DR copy" \
  --region us-west-2
```

---

## AWS Backup

```bash
# Create backup plan
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "EKS-DR-Plan",
  "Rules": [{
    "RuleName": "DailyDR",
    "TargetBackupVaultName": "Default",
    "ScheduleExpression": "cron(0 1 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 180,
    "RecoveryPointTags": {"purpose": "dr"}
  }]
}'
```

---

## Infrastructure as Code

### Terraform for DR

```hcl
# Create cluster in DR region
resource "aws_eks_cluster" "dr" {
  provider  = aws.dr_region
  name      = "eks-dr"
  role_arn  = aws_iam_role.eks_cluster.arn
  version   = "1.32"

  vpc_config {
    subnet_ids         = module.vpc.private_subnets
    endpoint_private_access = true
    endpoint_public_access  = false
  }

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }
}
```

### GitOps for DR

Argo CD ApplicationSet can deploy to multiple clusters:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
spec:
  generators:
    - list:
        elements:
          - cluster: prod-us-east-1
            url: https://kubernetes.default.svc
          - cluster: dr-us-west-2
            url: https://dr-cluster.example.com
  template:
    metadata:
      name: "{{cluster}}-my-app"
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/apps.git
        targetRevision: HEAD
        path: apps/my-app
      destination:
        server: "{{url}}"
        namespace: production
      syncPolicy:
        automated: {}
```

---

## RTO/RPO Example Configuration

| Component | Backup Method | RPO | RTO |
|-----------|--------------|-----|-----|
| Kubernetes resources | Velero → S3 | 24h | 30 min |
| EBS volumes | Snapshot lifecycle | 24h | < 1 hr |
| EC2 node config | Terraform (IaC) | N/A | 15 min |
| EFS | AWS Backup | 12h | < 1 hr |
| RDS | Automated + manual snapshots | 15 min | < 1 hr |
| ECR images | Cross-region replication | ~15 min | Immediate |
| Cluster config | GitOps (Argo CD) | N/A | Re-sync |

---

## DR Runbook Steps

1. **Initial Response**: Declare DR event, notify on-call
2. **Assess**: Determine scope of outage
3. **Restore Data**: Restore EBS volumes/RDS from snapshots
4. **Provision Cluster**: Run Terraform for DR region
5. **Deploy Workloads**: Argo CD syncs Git to DR cluster
6. **Restore Data**: Velero restores application data
7. **Switch Traffic**: Update Route 53 (failover routing policy) to point to DR ALB
8. **Validate**: Run health checks, verify application endpoints
9. **Post-DR**: Document lessons learned, update runbook

---

## Failback Strategy

1. Provision primary region infrastructure (standard state)
2. Sync data back (RDS replica, EBS snapshot copy)
3. Deploy to primary cluster
4. Validate primary
5. Switch traffic back
6. Decommission DR resources

---

## Testing and Validation

- **Quarterly DR drills**: restore backups and verify
- **Chaos engineering**: kill AZs, nodes, pods to validate resilience
- **Load testing**: ensure DR region can handle production traffic
- **Backup integrity**: test restore of backups, not just backup creation

---

## References

- [Velero](https://velero.io/docs/)
- [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [EKS Disaster Recovery](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [Kubernetes DR](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)