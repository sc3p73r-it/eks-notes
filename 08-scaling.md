# 8. EKS Scaling

## Overview

EKS provides multiple scaling mechanisms: pod-level scaling (HPA, VPA), node-level scaling (Cluster Autoscaler, Karpenter), and infrastructure-level scaling (Auto Scaling Groups, EKS Auto Mode). Understanding when to use each is critical for cost and performance optimization.

---

## Scaling Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         EKS Scaling Layers                               │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Pod Scaling                                                      │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │  │
│  │  │   HPA    │  │   VPA    │  │ CronHPA  │  │ KEDA     │        │  │
│  │  │ (pods)   │  │ (resize) │  │ (schedule)│  │ (events) │        │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Node Scaling                                                     │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                      │  │
│  │  │Cluster Autoscaler│  │    Karpenter      │                      │  │
│  │  │  (ASG-based)     │  │  (Direct EC2)     │                      │  │
│  │  └──────────────────┘  └──────────────────┘                      │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Infrastructure Scaling                                           │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                      │  │
│  │  │  ASG Scaling     │  │  EKS Auto Mode   │                      │  │
│  │  │  (min/max/des)   │  │  (Auto)          │                      │  │
│  │  └──────────────────┘  └──────────────────┘                      │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Horizontal Pod Autoscaler (HPA)

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    HPA Flow                                   │
│                                                               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ Metrics  │───▶│  HPA     │───▶│ReplicaSet│──▶ Pods       │
│  │ Server   │    │Controller│    │          │               │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                               │
│  Metrics: CPU, Memory, Custom (Prometheus), External (SQS)  │
└─────────────────────────────────────────────────────────────┘
```

### Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### HPA with Custom Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 1000
    - type: External
      external:
        metric:
          name: sqs_queue_length
          selector:
            matchLabels:
              queue: my-queue
        target:
          type: AverageValue
          averageValue: 30
```

---

## Vertical Pod Autoscaler (VPA)

### Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| Off | Recommendation only | Analysis |
| Initial | Apply on pod creation | New deployments |
| Auto | Apply on creation and updates | Continuous optimization |
| Recreate | Restart pods to apply | Stateful workloads |

### Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
        controlledResources: ["cpu", "memory"]
```

---

## Cluster Autoscaler

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Cluster Autoscaler                         │
│                                                               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ Pending  │───▶│  Scale   │───▶│   ASG    │──▶ EC2        │
│  │   Pods   │    │  Up/Down │    │          │               │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                               │
│  Triggers:                                                    │
│  - Pods pending due to insufficient resources                 │
│  - Nodes underutilized for 10+ minutes                        │
│  - No pods scheduled on node for 10+ minutes                 │
└─────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# Install via Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1
```

### Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
```

### Expander Options

| Type | Description |
|------|-------------|
| random | Random selection |
| most-pods | Scale to node with most pending pods |
| least-waste | Scale to node with least wasted resources |
| priority | Use priority-based expander |

---

## Karpenter

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Karpenter                                  │
│                                                               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ Pending  │───▶│  Node    │───▶│   EC2    │──▶ Pods       │
│  │   Pods   │    │Provisioner│   │ Instance │               │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                               │
│  Features:                                                    │
│  - Direct EC2 provisioning (no ASG)                          │
│  - Instance type selection                                   │
│  - Spot support                                               │
│  - Node consolidation                                         │
│  - Expiration                                                │
└─────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# Install via Helm
helm repo add karpenter https://karpenter.sh/oci-charts
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set settings.clusterName=my-cluster \
  --set settings.interruptionHandling=true
```

### NodePool

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["m", "c", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        name: default
  limits:
    cpu: "1000"
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

### EC2NodeClass

```yaml
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  role: KarpenterNodeRole-my-cluster
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        encrypted: true
```

---

## Cluster Autoscaler vs Karpenter

| Feature | Cluster Autoscaler | Karpenter |
|---------|-------------------|-----------|
| Provisioning | Via ASG | Direct EC2 |
| Speed | 5-10 minutes | ~60 seconds |
| Instance selection | ASG-defined | Any instance type |
| Spot support | Via ASG | Native |
| Consolidation | No | Yes |
| Flexibility | Limited | High |
| Overhead | ASG costs | Lower |
| Recommended | Predictable workloads | Dynamic workloads |

---

## Pod Disruption Budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

---

## Scaling Best Practices

| Area | Recommendation |
|------|----------------|
| HPA | Set resource requests/limits, use custom metrics |
| VPA | Use in recommendation mode first |
| Karpenter | Use consolidation for cost savings |
| PDB | Always define for critical workloads |
| Cluster Autoscaler | Use for predictable workloads |
| Monitoring | Track scaling events, set alerts |

---

## Troubleshooting

```bash
# Check HPA status
kubectl get hpa
kubectl describe hpa <hpa-name>

# Check Cluster Autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100

# Check Karpenter logs
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --tail=100

# Check scaling events
kubectl get events --field-selector reason=ScalingReplicaSet

# Check node capacity
kubectl describe nodes | grep -A5 "Allocated resources"
```

---

## References

- [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [Karpenter](https://karpenter.sh/docs/)
- [EKS Best Practices - Scaling](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
