# 26. EKS Application Deployment Example

## Overview

This section demonstrates a complete production application deployment on EKS: a frontend, backend API, worker, and PostgreSQL database, with all supporting Kubernetes and AWS resources.

---

## Application Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Sample E-Commerce Application                     │
│                                                                     │
│                     ┌──────────────┐                                │
│           ──────▶   │   Frontend    │                               │
│    Internet / ALB   │   (React)     │                               │
│                     └──────┬───────┘                                │
│                            │                                        │
│                     ┌──────▼───────┐     ┌─────────────────┐       │
│                     │  Backend API  │────▶│  Worker (Queue) │       │
│                     │  (Node.js)    │     │  (Python)       │       │
│                     └──────┬───────┘     └────────┬────────┘       │
│                            │                       │                │
│                     ┌──────▼───────┐     ┌────────▼────────┐       │
│                     │  PostgreSQL   │     │   SQS Queue     │       │
│                     │  (RDS / Stateful)│   │   ElastiCache   │       │
│                     └──────────────┘     └─────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    env: production
    pod-security.kubernetes.io/enforce: restricted
```

```bash
kubectl apply -f 01-namespace.yaml
```

---

## 2. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ecommerce
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "500"
  FEATURE_FLAGS: "cart,checkout"
  DATABASE_NAME: "ecommerce"
  DATABASE_HOST: "ecommerce-db-postgresql.ecommerce.svc.cluster.local"
```

---

## 3. Secret (via External Secrets / Secrets Manager)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: ecommerce
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-secrets
  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: prod/ecommerce/database
        property: password
    - secretKey: API_KEY
      remoteRef:
        key: prod/ecommerce/api
        property: api_key
```

---

## 4. ServiceAccount + IRSA/Pod Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: ecommerce
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ecommerce-backend-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-access
  namespace: ecommerce
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-role-binding
  namespace: ecommerce
subjects:
  - kind: ServiceAccount
    name: backend-sa
    namespace: ecommerce
roleRef:
  kind: Role
  name: backend-access
  apiGroup: rbac.authorization.k8s.io
```

---

## 5. Frontend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ecommerce-frontend:v1.2.3
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

---

## 6. Frontend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

---

## 7. Backend API Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: ecommerce
  labels:
    app: backend
spec:
  replicas: 5
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      serviceAccountName: backend-sa
      containers:
        - name: backend
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ecommerce-backend:v1.2.3
          env:
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_PASSWORD
            - name: DATABASE_HOST
              value: ecommerce-db.ecommerce.svc.cluster.local
            - name: REDIS_URL
              value: redis://elasticache-redis.example.com:6379
            - name: SQS_QUEUE_URL
              value: https://sqs.us-east-1.amazonaws.com/123456789012/ecommerce-orders
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
```

---

## 8. Backend Service (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: ecommerce
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

---

## 9. Worker Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
  namespace: ecommerce
  labels:
    app: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      serviceAccountName: backend-sa
      containers:
        - name: worker
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ecommerce-worker:v1.2.3
          env:
            - name: SQS_QUEUE_URL
              value: https://sqs.us-east-1.amazonaws.com/123456789012/ecommerce-orders
            - name: DATABASE_HOST
              value: ecommerce-db.ecommerce.svc.cluster.local
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
```

---

## 10. Database (PostgreSQL) via StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-db
  namespace: ecommerce
  labels:
    app: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ecommerce-db
  namespace: ecommerce
spec:
  serviceName: ecommerce-db
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_DB
              value: ecommerce
            - name: POSTGRES_USER
              value: ecommerce
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_PASSWORD
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: "2"
              memory: 2Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 100Gi
```

**Production note**: For critical databases, consider RDS/Aurora instead of in-cluster StatefulSet (managed backups, patching, multi-AZ).

---

## 11. Ingress (ALB)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc-123
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:123456789012:regional/webacl/ecommerce/abc-123
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

---

## 12. HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: External
      external:
        metric:
          name: sqs_queue_length
          selector:
            matchLabels:
              queue: ecommerce-orders
        target:
          type: AverageValue
          averageValue: 100
```

---

## 13. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: ecommerce
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: frontend
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
  namespace: ecommerce
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: backend
```

---

## 14. Communication with AWS Services

### SQS (via IRSA)

```yaml
# IAM policy (attached to backend-sa role)
apiVersion: iam.aws.amazon.com/v1
kind: Policy
metadata:
  name: sqs-policy
spec:
  statements:
    - Effect: Allow
      Action:
        - sqs:SendMessage
        - sqs:ReceiveMessage
      Resource: arn:aws:sqs:us-east-1:123456789012:ecommerce-orders
```

### ElastiCache (via Security Group)

- ElastiCache is accessed over its cluster endpoint from pods
- Pods need egress to ElastiCache security group (update the node security group)

---

## Apply Sequence

```bash
# 1. Create namespace
kubectl apply -f 01-namespace.yaml

# 2. Create ConfigMap and Secret
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-secret.yaml

# 3. Create ServiceAccounts + RBAC
kubectl apply -f 04-serviceaccount.yaml

# 4. Create database (StatefulSet) first due to dependency
kubectl apply -f 10-database.yaml

# 5. Deploy application workloads
kubectl apply -f 05-frontend-deployment.yaml
kubectl apply -f 06-frontend-service.yaml
kubectl apply -f 07-backend-deployment.yaml
kubectl apply -f 08-backend-service.yaml
kubectl apply -f 09-worker-deployment.yaml

# 6. Ingress
kubectl apply -f 11-ingress.yaml

# 7. Scaling and PDB
kubectl apply -f 12-hpa.yaml
kubectl apply -f 13-pdb.yaml

# 8. Verify
kubectl get all -n ecommerce
kubectl get ingress -n ecommerce
kubectl get hpa -n ecommerce
kubectl get pdb -n ecommerce
```

---

## Verification

```bash
# Pods running
kubectl get pods -n ecommerce -o wide

# Services reachable
kubectl exec -it <frontend-pod> -n ecommerce -- curl http://backend

# Database connectivity
kubectl exec -it <backend-pod> -n ecommerce -- \
  psql "postgresql://ecommerce:PASSWORD@ecommerce-db.ecommerce.svc.cluster.local/ecommerce" -c "SELECT 1;"

# Ingress traffic
curl -H "Host: shop.example.com" https://<ALB-DNS>/healthz

# HPA status
kubectl get hpa backend-hpa -n ecommerce

# Events
kubectl get events -n ecommerce --sort-by=.metadata.creationTimestamp
```

---

## Production Considerations

| Area | Recommendation |
|------|----------------|
| Databases | Prefer RDS/Aurora for production; EBS StatefulSet for dev/PoC |
| Secrets | Use External Secrets Operator, never plain K8s Secrets in Git |
| Images | Immutable tags (git SHA), never `latest` in prod |
| Resources | Set requests/limits to allow HPA and scheduling |
| PDB | Always define for critical components |
| Monitoring | Container Insights + Prometheus + Grafana |
| NetworkPolicy | Enable default-deny per namespace |
| Backup | Velero + EBS snapshots + AWS Backup for RDS/EFS |
| GitOps | Manage all above manifests via Argo CD |

---

## References

- [Kubernetes Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [AWS Load Balancer Controller](https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html)
- [External Secrets](https://external-secrets.io/)
- [StatefulSet on EKS](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)