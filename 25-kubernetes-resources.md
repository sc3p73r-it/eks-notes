# 25. Kubernetes Resources Inside EKS

## Overview

This section explains the key Kubernetes resources used in EKS, including their purpose, architecture, example use case, and production considerations.

---

## Workload Resources

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    pod-security.kubernetes.io/enforce: restricted
```

| Aspect | Detail |
|--------|--------|
| Purpose | Isolate resources, quotas, RBAC, network policies |
| Use case | Environment per namespace (dev/staging/prod), team isolation |
| Production | Set resource quotas, apply Pod Security Standards |

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  restartPolicy: Always
  securityContext:
    runAsNonRoot: true
  containers:
    - name: app
      image: my-app:v1
      resources:
        requests: { cpu: 100m, memory: 128Mi }
```

| Aspect | Detail |
|--------|--------|
| Purpose | Smallest compute unit, wraps one or more containers |
| Use case | Direct pod management (usually via controllers) |
| Production | Use Deployments/StatefulSets, not raw Pods |

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/web:v1.2.3
```

| Aspect | Detail |
|--------|--------|
| Purpose | Declarative rollout/rollback of ReplicaSets |
| Use case | Stateless applications |
| Production | Set strategy (RollingUpdate) with maxSurge/maxUnavailable |

### ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-5b7d8f9c6
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    spec:
      containers:
        - name: web
          image: web:v1.2.3
```

| Aspect | Detail |
|--------|--------|
| Purpose | Maintain stable number of identical pod replicas |
| Use case | Managed by Deployments; rarely created directly |
| Production | Let Deployment manage ReplicaSet |

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: production
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
        - name: db
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 50Gi
```

| Aspect | Detail |
|--------|--------|
| Purpose | Stable identity, stable storage, ordered deployment |
| Use case | Databases, queues with ordering requirements |
| Production | Use EBS/EFS carefully; PDB per pod identity |

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
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:latest
```

| Aspect | Detail |
|--------|--------|
| Purpose | Runs one pod per node |
| Use case | Log agents, monitoring agents, node exporters |
| Production | Runs kube-proxy (as DaemonSet via EKS), VPC CNI aws-node, Fluent Bit |

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: my-app.jobs:migrate-v1
```

| Aspect | Detail |
|--------|--------|
| Purpose | Run to completion |
| Use case | DB migrations, one-off scripts |
| Production | Set backoffLimit, TTL-after-finish |

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: my-cleanup
          restartPolicy: OnFailure
```

| Aspect | Detail |
|--------|--------|
| Purpose | Scheduled jobs |
| Use case | Report generation, cleanup, scheduled operations |
| Production | Use timezone config, monitor missed schedules |

---

## Service & Network Resources

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

| Aspect | Detail |
|--------|--------|
| Purpose | Stable virtual IP + DNS for a set of pods |
| Use case | Service discovery between apps |
| Production | Use ClusterIP for internal, LoadBalancer for external |

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port: { number: 80 }
```

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  config.yaml: |
    log_level: info
    max_conns: 100
```

| Aspect | Detail |
|--------|--------|
| Purpose | Non-confidential configuration |
| Use case | YAML/JSON config, env vars |
| Production | Use `kubectl create configmap --from-file` for config files |

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  password: cGFzc3dvcmQ=
```

See also [Secrets Management](17-secrets-management.md).

---

## Access Control Resources

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/app-role
```

### Role / RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: developer
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole / ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
  - kind: Group
    name: operators
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

| Aspect | Detail |
|--------|--------|
| Purpose | Restrict pod-to-pod traffic |
| Use case | Segmentation, security |
| Production | Requires CNI support (VPC CNI with network policy, Calico, Cilium) |

---

## Storage Resources

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### PersistentVolume (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0123456789abcdef0
  claimRef:
    namespace: production
    name: my-pvc
```

### PersistentVolumeClaim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
```

---

## Autoscaling Resources

### HorizontalPodAutoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### PodDisruptionBudget (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

---

## Resource Decision Summary

| Workload need | Resource |
|---------------|----------|
| Stateless app | Deployment |
| Stateful app with storage | StatefulSet |
| Run one per node | DaemonSet |
| One-off task | Job |
| Scheduled task | CronJob |
| Stable network identity | Service |
| External HTTP access | Ingress + ALB |
| Configuration | ConfigMap |
| Sensitive data | Secret + External Secrets |
| Non-root permissions | ServiceAccount + IRSA/Pod Identity |
| Restrict API access | RBAC |
| Restrict traffic | NetworkPolicy |
| Block storage | StorageClass + PVC |
| Pod autoscaling | HPA |
| Reduce disruption | PDB |

---

## References

- [Kubernetes Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)