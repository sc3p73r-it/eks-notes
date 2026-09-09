# 11. EKS Monitoring and Alerting

## Overview

Production monitoring for EKS requires visibility into the cluster, nodes, pods, and applications. A comprehensive monitoring strategy combines Kubernetes-native metrics, AWS CloudWatch metrics, Prometheus, and alerting that routes to the right teams.

---

## Monitoring Dimensions

| Dimension | What to Monitor | Tool |
|-----------|-----------------|------|
| Cluster | API server, etcd, scheduler | CloudWatch, Prometheus |
| Node | CPU, memory, disk, network | Container Insights |
| Pod | Restart count, OOMKilled | Prometheus, CloudWatch |
| Application | Request rate, latency, errors | Prometheus, X-Ray |
| Security | Failed auth, RBAC denials | GuardDuty, CloudTrail |

---

## Recommended Alerts

### Cluster Level

| Alert | Condition | Severity |
|-------|-----------|----------|
| API Server Unavailable | 5xx errors > 1% for 5 min | Critical |
| etcd Latency High | etcd request latency > 100ms | Warning |
| Node NotReady | Node NotReady for 5 min | Critical |
| Pod Scheduling Failed | Unschedulable pods > 0 for 10 min | Warning |

### Node Level

| Alert | Condition | Severity |
|-------|-----------|----------|
| High CPU | Node CPU > 90% for 10 min | Warning |
| High Memory | Node memory > 90% for 10 min | Warning |
| Disk Pressure | Node disk usage > 85% | Warning |
| Network Unavailable | Node network unavailable | Critical |

### Pod Level

| Alert | Condition | Severity |
|-------|-----------|----------|
| CrashLoopBackOff | Pod restarts > 5 in 10 min | Warning |
| OOMKilled | Container OOM killed | Warning |
| Pod Pending | Pod pending > 15 min | Warning |
| High Error Rate | 5xx errors > 5% for 5 min | Critical |

---

## CloudWatch Alarms

### Node CPU Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-High-CPU" \
  --alarm-description "CPU utilization is high" \
  --metric-name CPUUtilized \
  --namespace ContainerInsights \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions "Name=ClusterName,Value=my-cluster" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-topic
```

### Pod Restart Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-Pod-Restarts" \
  --alarm-description "Pod restarts are high" \
  --metric-name pod_number_of_container_restarts \
  --namespace ContainerInsights \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions "Name=ClusterName,Value=my-cluster" "Name=Namespace,Value=production" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-topic
```

### Memory Utilization Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-High-Memory" \
  --alarm-description "Memory utilization is high" \
  --metric-name MemoryUtilized \
  --namespace ContainerInsights \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions "Name=ClusterName,Value=my-cluster" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-topic
```

---

## Prometheus AlertRules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: eks-alerts
  namespace: monitoring
spec:
  groups:
    - name: cluster-alerts
      rules:
        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} is not ready"

        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"

        - alert: HighMemoryUsage
          expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage on {{ $labels.instance }}"

        - alert: PodPending
          expr: kube_pod_status_phase{phase="Pending"} == 1
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} pending for too long"

        - alert: APIHighLatency
          expr: histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m])) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "API server latency is high"
```

---

## Key Metrics to Monitor

### Kubernetes Metrics

```bash
# Check node resource usage
kubectl top nodes

# Check pod resource usage sorted by CPU
kubectl top pods --sort-by=cpu

# Check pod resource usage sorted by memory
kubectl top pods --sort-by=memory

# Check API server request latency
kubectl get --raw /metrics | grep apiserver_request_duration_seconds_bucket

# Check scheduler metrics
kubectl get --raw /metrics | grep scheduler_e2e_scheduling_duration_seconds_bucket

# Check container restarts
kubectl get pods -A | grep -i crashloopbackoff
kubectl describe pod <pod-name> | grep -A3 "Restart Count"
```

### CloudWatch Container Insights Metrics

| Metric | Description |
|--------|-------------|
| `ClusterNodeCount` | Number of nodes in cluster |
| `ClusterFailedNodeCount` | Number of failed nodes |
| `NodeCpuUtilized` | Node CPU utilization |
| `pod_cpu_utilization` | Pod CPU utilization |
| `pod_memory_utilization` | Pod memory utilization |
| `pod_network_rx_bytes` | Pod network receive bytes |
| `pod_network_tx_bytes` | Pod network transmit bytes |
| `pod_number_container_restarts` | Container restart count |

---

## Alerting Routing

| Alert Severity | Channel | Response Time |
|----------------|---------|---------------|
| Critical | PagerDuty/Opsgenie | Immediate (24x7) |
| Warning | Slack/Teams | Business hours |
| Info | Slack notifications | Best effort |

### Alert Manager Configuration

```yaml
apiVersion: monitoring.coreos.com/v1
kind: AlertmanagerConfig
metadata:
  name: eks-alerts
  namespace: monitoring
spec:
  route:
    groupBy: ["alertname", "cluster"]
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    receiver: "slack-alerts"
    routes:
      - matchers:
          - name: severity
            value: critical
        receiver: "pagerduty-critical"
  receivers:
    - name: slack-alerts
      slackConfigs:
        - apiURL:
            key: webhook-url
            name: slack-secret
          channel: "#platform-alerts"
          text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}"
    - name: pagerduty-critical
      pagerDutyConfigs:
        - serviceKey:
            key: service-key
            name: pagerduty-secret
          severity: critical
```

---

## Dashboard Recommendations

| Dashboard | Metrics |
|-----------|---------|
| Cluster Overview | Node count, CPU/memory usage, pod count |
| Node Status | Node health, capacity, utilization |
| Pod Health | Restarts, OOM, pending, crashloop |
| Application | Request rate, latency, error rate |
| Networking | Pod network throughput, errors |

---

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| Monitoring | Use Container Insights + Prometheus together |
| Alerting | CloudWatch Alarms + Prometheus AlertManager |
| Dashboards | Grafana for custom dashboards |
| Log retention | Set appropriate retention policies |
| Cost | Use CloudWatch Contributor Insights, limit high-cardinality metrics |
| On-call | Integrate alerts with PagerDuty/Opsgenie |
| Runbooks | Document alert response procedures |
| Testing | Regularly test alerting (e.g., kill a node) |

---

## Additional Monitoring Commands

```bash
# Check pod events sorted by time
kubectl get events --sort-by=.metadata.creationTimestamp

# Check ALL pods in CrashLoopBackOff
kubectl get pods --all-namespaces | grep -i crash

# Check node conditions
kubectl get node <node-name> -o jsonpath='{range .status.conditions[*]}{.type}{" = "}{.status}{"\n"}{end}'

# Check kubelet logs on a node
kubectl logs --tail=100 <kubelet-pod> -n kube-system || journalctl -u kubelet --no-pager

# Get cluster summary
kubectl get nodes -o wide
kubectl get pods -A -o wide --sort-by=.status.phase

# Check resource quota usage
kubectl get resourcequota -A
kubectl describe resourcequota --all-namespaces
```

---

## References

- [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights.html)
- [Prometheus on EKS](https://docs.aws.amazon.com/eks/latest/best-practices/prometheus.html)
- [CloudWatch Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [Monitoring EKS](https://docs.aws.amazon.com/eks/latest/userguide/eks-observability.html)