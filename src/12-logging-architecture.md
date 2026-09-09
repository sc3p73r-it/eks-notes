# 12. EKS Logging Architecture

## Overview

EKS logging captures application logs (stdout/stderr), control plane logs, and audit logs. The standard pipeline uses Fluent Bit as a DaemonSet to collect container logs and ship them to CloudWatch Logs or other destinations.

---

## Log Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EKS Logging Architecture                               │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Application Logs                                                 │  │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │  │
│  │  │ Container│───▶│Fluent Bit│───▶│CloudWatch│───▶│   S3     │  │  │
│  │  │ stdout/  │    │(DaemonSet│    │  Logs    │    │(archive) │  │  │
│  │  │ stderr   │    │          │    │          │    │          │  │  │
│  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Control Plane Logs                                               │  │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │  │
│  │  │  EKS API │───▶│CloudWatch│───▶│OpenSearch│                   │  │
│  │  │  Server  │    │  Logs    │    │(optional)│                   │  │
│  │  └──────────┘    └──────────┘    └──────────┘                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Fluent Bit Configuration

### DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
        - name: fluent-bit
          image: public.ecr.aws/aws-observability/aws-for-fluent-bit:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off

    [OUTPUT]
        Name                cloudwatch_logs
        Match               *
        region              us-east-1
        log_group_name      /eks/my-cluster
        log_stream_prefix   fluentbit-
        auto_create_group   true

  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Decode_Field_As json log
```

---

## Structured Logging

### JSON Log Format

```json
{
  "timestamp": "2025-01-15T10:30:00.000Z",
  "level": "info",
  "message": "Request processed",
  "service": "api-gateway",
  "method": "GET",
  "path": "/api/v1/users",
  "status": 200,
  "duration_ms": 45,
  "correlation_id": "abc-123-def-456",
  "user_id": "user-789"
}
```

### Correlation IDs

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            - name: CORRELATION_ID_HEADER
              value: "X-Correlation-ID"
            - name: SERVICE_NAME
              value: "my-app"
```

---

## Log Groups and Retention

```bash
# Create log group
aws logs create-log-group --log-group-name /eks/my-cluster --retention-in-days 30

# Update retention
aws logs put-retention-policy --log-group-name /eks/my-cluster --retention-in-days 90
```

### Recommended Retention

| Log Type | Retention | Cost |
|----------|-----------|------|
| Application logs | 30-90 days | Medium |
| Control plane logs | 90-365 days | High |
| Audit logs | 365+ days | High |
| Security logs | Indefinite | High |

---

## OpenSearch Integration

```yaml
# Fluent Bit output to OpenSearch
[OUTPUT]
    Name                opensearch
    Match               *
    Host                my-opensearch-endpoint.us-east-1.es.amazonaws.com
    Port                443
    AWS_Auth            On
    AWS_Region          us-east-1
    Index               eks-logs
    Type                _doc
    Logstash_Format     On
    Logstash_Prefix     eks-logs
    Logstash_DateFormat %Y.%m.%d
```

---

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| Format | Use structured JSON logging |
| Correlation | Include correlation IDs in all services |
| Retention | Set appropriate retention policies |
| Cost | Archive old logs to S3 |
| Security | Enable control plane logging |
| Monitoring | Set up CloudWatch Insights queries |

---

## Troubleshooting

```bash
# Check Fluent Bit pods
kubectl get pods -n logging -l app=fluent-bit

# Check Fluent Bit logs
kubectl logs -n logging -l app=fluent-bit --tail=100

# Query CloudWatch Logs Insights
aws logs start-query \
  --log-group-name /eks/my-cluster \
  --start-time $(date -d '1 hour ago' +%s000) \
  --end-time $(date +%s000) \
  --query-string "fields @timestamp, @message | filter @message like /ERROR/ | limit 20"
```

---

## References

- [EKS Control Plane Logging](https://docs.aws.amazon.com/eks/latest/userguide/logging.html)
- [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup.html)
- [Fluent Bit](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-logs-fluentbit.html)
