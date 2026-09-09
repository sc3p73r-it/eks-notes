# 13. EKS Control Plane Logging

## Overview

EKS control plane logging captures logs from the Kubernetes API server, audit, authenticator, controller manager, and scheduler. These logs are sent to Amazon CloudWatch Logs and are essential for security auditing, troubleshooting, and compliance.

---

## Log Types

| Log Type | Description | Use Case |
|----------|-------------|----------|
| `api` | Kubernetes API server logs | API request/response details |
| `audit` | Kubernetes audit logs | Security auditing, compliance |
| `authenticator` | EKS authenticator logs | IAM authentication debugging |
| `controllerManager` | Kubernetes controller manager logs | Controller debugging |
| `scheduler` | Kubernetes scheduler logs | Pod scheduling debugging |

---

## Enable Control Plane Logging

```bash
# Enable all log types
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Enable specific log types
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit"],"enabled":true}]}'

# Verify logging status
aws eks describe-cluster --name my-cluster --query "cluster.logging"
```

---

## Log Storage

Control plane logs are stored in CloudWatch Logs:

```
/aws/eks/my-cluster/cluster
├── api/
├── audit/
├── authenticator/
├── controllerManager/
└── scheduler/
```

---

## Audit Log Analysis

```sql
# CloudWatch Logs Insights - Find failed authentication
fields @timestamp, @message
| filter @message like /"verb":"get"/
| filter @message like /"code":403/
| limit 20

# Find API requests by user
fields @timestamp, @message
| parse @message '"user": {"username": "*"}' as username
| stats count(*) as requests by username
| sort requests desc

# Find resource creation events
fields @timestamp, @message
| filter @message like /"verb":"create"/
| parse @message '"resource": "*"' as resource
| stats count(*) as creates by resource
| sort creates desc
```

---

## Cost Considerations

| Log Type | Volume | Cost Impact |
|----------|--------|-------------|
| API | Medium | Medium |
| Audit | High | High |
| Authenticator | Low | Low |
| Controller Manager | Low | Low |
| Scheduler | Low | Low |

**Recommendation**: Enable at minimum `api` and `audit` for security and compliance. Enable all types only during troubleshooting.

---

## Security Considerations

- Audit logs may contain sensitive information (usernames, resource names)
- Enable CloudWatch Logs encryption with KMS
- Set appropriate log retention policies
- Use CloudWatch Logs Insights for analysis
- Export to S3 for long-term archival

```bash
# Enable CloudWatch Logs encryption
aws logs put-key-policy \
  --key-alias alias/eks-logs \
  --policy-name eks-logs-policy \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "logs.us-east-1.amazonaws.com"},
      "Action": "kms:Decrypt",
      "Resource": "*"
    }]
  }'
```

---

## Troubleshooting

```bash
# Check control plane logs
aws logs describe-log-groups --log-group-name-prefix /aws/eks/my-cluster

# Get recent logs
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/cluster \
  --start-time $(date -d '1 hour ago' +%s000) \
  --filter-pattern "userInfo.username"

# Get authenticator logs
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/cluster \
  --filter-pattern "authenticator"

# Check audit logs for specific user
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/cluster \
  --filter-pattern '[$.user.username = "admin"]'
```

---

## References

- [Control Plane Logging](https://docs.aws.amazon.com/eks/latest/userguide/logging.html)
- [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
