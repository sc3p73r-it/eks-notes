# 10. EKS Observability

## Overview

Observability in EKS encompasses metrics, logs, and traces. AWS provides native integrations through CloudWatch, Container Insights, and X-Ray. Third-party tools like Prometheus, Grafana, and Jaeger are also commonly used.

---

## Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    EKS Observability Stack                        │
│                                                                   │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │    Metrics     │  │     Logs      │  │    Traces     │        │
│  │               │  │               │  │               │        │
│  │  Prometheus   │  │  Fluent Bit   │  │  X-Ray        │        │
│  │  CloudWatch   │  │  CloudWatch   │  │  OpenTelemetry│        │
│  │  Container    │  │  OpenSearch   │  │  Jaeger       │        │
│  │  Insights     │  │  S3           │  │               │        │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘        │
│          │                  │                  │                  │
│          └──────────────────┼──────────────────┘                  │
│                             ▼                                     │
│                    ┌───────────────┐                              │
│                    │   Grafana     │                              │
│                    │  (Dashboards) │                              │
│                    └───────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Metrics

### Container Insights

```bash
# Install Container Insights
ClusterName=<cluster-name>
RegionName=<region>

curl https://raw.githubusercontent.com/aws-observability/aws-observability-accelerator/main/artifacts/container-insights/fluent-bit.yaml | \
  sed "s/your-cluster-name/${ClusterName}/g; s/your-cluster-region/${RegionName}/g" | \
  kubectl apply -f -
```

### Key Metrics

| Metric | Description | Source |
|--------|-------------|--------|
| `ContainerInsights/PodStatus` | Pod state | Container Insights |
| `ContainerInsights/CpuUtilized` | CPU utilization | Container Insights |
| `ContainerInsights/MemoryUtilized` | Memory utilization | Container Insights |
| `ContainerInsights/NetworkRxBytes` | Network receive | Container Insights |
| `ContainerInsights/NetworkTxBytes` | Network transmit | Container Insights |
| `pod_cpu_utilization` | Pod CPU | Prometheus |
| `pod_memory_working_set` | Pod memory | Prometheus |
| `http_requests_total` | HTTP requests | Prometheus |

### Prometheus

```bash
# Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin
```

### ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## Logs

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Log Pipeline                                  │
│                                                                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │Container │───▶│Fluent Bit│───▶│CloudWatch│───▶│OpenSearch│  │
│  │ stdout/  │    │  (DaemonSet)│ │  Logs    │    │          │  │
│  │ stderr   │    │           │    │          │    │          │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### CloudWatch Logs Insights

```sql
# Find high CPU pods
fields @timestamp, kubernetes.pod_name, kubernetes.namespace
| filter type = "Pod"
| stats avg(pod_cpu_utilization) as avg_cpu by kubernetes.pod_name, kubernetes.namespace
| sort avg_cpu desc
| limit 10

# Find pods using most memory
fields @timestamp, kubernetes.pod_name, kubernetes.namespace
| filter type = "Pod"
| stats avg(pod_memory_utilization) as avg_mem by kubernetes.pod_name, kubernetes.namespace
| sort avg_mem desc
| limit 10
```

---

## Tracing

### AWS X-Ray

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
          image: my-app:latest
          env:
            - name: AWS_XRAY_DAEMON_ADDRESS
              value: "xray-service:2000"
      sidecars:
        - name: xray-daemon
          image: public.ecr.aws/xray/aws-xray-daemon:latest
          ports:
            - containerPort: 2000
              protocol: UDP
```

### OpenTelemetry (ADOT)

```bash
# Install ADOT Operator
kubectl apply -f https://github.com/aws-observability/aws-otel-collector/releases/latest/download/otel-otel-operator.yaml
```

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: observability
spec:
  mode: deployment
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
    exporters:
      awsxray:
        region: us-east-1
      awsemf:
        region: us-east-1
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsxray]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsemf]
```

---

## Dashboards

### Grafana

```bash
# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword=admin \
  --set persistence.enabled=true
```

### Amazon Managed Grafana

```bash
# Create workspace
aws grafana create-workspace \
  --workspace-name my-grafana \
  --authentication-method AWS_SSO \
  --permission-type SERVICE_MANAGED \
  --data-sources CLOUDWATCH \
  --region us-east-1
```

---

## Production Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Production Observability                       │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Metrics Pipeline                                         │   │
│  │  Prometheus → AMP → Grafana Dashboards                    │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Logs Pipeline                                            │   │
│  │  Fluent Bit → CloudWatch Logs → S3 (archive)              │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Traces Pipeline                                          │   │
│  │  ADOT Collector → X-Ray / AMP                             │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Alerting                                                 │   │
│  │  Prometheus AlertManager → SNS → PagerDuty / Slack        │   │
│  └───────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## References

- [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup.html)
- [Prometheus](https://docs.aws.amazon.com/eks/latest/best-practices/prometheus.html)
- [X-Ray](https://docs.aws.amazon.com/eks/latest/best-practices/tracing.html)
- [OpenTelemetry](https://docs.aws.amazon.com/eks/latest/best-practices/opentelemetry.html)
