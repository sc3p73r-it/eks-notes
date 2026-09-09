# 5. EKS IAM and Security

## Overview

EKS security is a shared responsibility model. AWS secures the control plane; the customer secures the data plane, workloads, and access. This section covers IAM integration, authentication, authorization, pod IAM, security groups, and Kubernetes security.

---

## Authentication and Authorization Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EKS Authentication Flow                               │
│                                                                         │
│  1. kubectl apply -f deployment.yaml                                    │
│           │                                                             │
│           ▼                                                             │
│  2. kubectl → AWS STS (GetCallerIdentity / AssumeRole)                 │
│           │                                                             │
│           ▼                                                             │
│  3. AWS STS → Returns IAM identity (user/role ARN)                     │
│           │                                                             │
│           ▼                                                             │
│  4. kubectl → EKS API Server (with Bearer token from STS)             │
│           │                                                             │
│           ▼                                                             │
│  5. EKS API Server → Authenticator validates token                     │
│           │   Maps IAM identity to Kubernetes identity                  │
│           ▼                                                             │
│  6. EKS API Server → RBAC authorization check                          │
│           │   Checks ClusterRole/Role/RoleBinding                      │
│           ▼                                                             │
│  7. EKS API Server → Admission controllers                              │
│           │   Pod Security, Resource Quotas, etc.                       │
│           ▼                                                             │
│  8. Request processed, resource created                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## EKS Access Entries (Recommended)

EKS access entries provide a simplified way to manage cluster access without the aws-auth ConfigMap:

```bash
# Create access entry for an IAM role
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/cluster-admin-role \
  --type STANDARD

# Associate access policy
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/cluster-admin-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

# List access entries
aws eks list-access-entries --cluster-name my-cluster

# Describe access entry
aws eks describe-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/cluster-admin-role
```

### Access Policies

| Policy | Scope | Description |
|--------|-------|-------------|
| `AmazonEKSClusterAdminPolicy` | Cluster | Full cluster admin |
| `AmazonEKSEditPolicy` | Cluster | Edit resources (no RBAC) |
| `AmazonEKSCloudWatchPolicy` | Cluster | CloudWatch access |
| `AmazonEKSServicePolicy` | Cluster | Service-linked roles |
| `AmazonEKSAdminPolicy` | Cluster | Admin with node access |

---

## aws-auth ConfigMap (Legacy)

The aws-auth ConfigMap maps IAM identities to Kubernetes RBAC. While still functional, EKS access entries are the recommended replacement:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/eks-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/admin-role
      username: admin
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/developer1
      username: developer1
      groups:
        - developers
```

---

## Kubernetes RBAC

```yaml
# Namespace-scoped Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: developer1
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
# ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/status"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
  - kind: Group
    name: system:masters
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## EKS Pod Identity

EKS Pod Identity is the recommended way to grant AWS permissions to pods:

```bash
# Create IAM role for pods
aws iam create-role \
  --role-name MyPodRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }'

# Attach policy to role
aws iam attach-role-policy \
  --role-name MyPodRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create Pod Identity association
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace production \
  --service-account my-service-account \
  --role-arn arn:aws:iam::123456789012:role/MyPodRole
```

### Pod Identity Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
```

---

## IRSA (IAM Roles for Service Accounts)

```bash
# Create OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# Create service account with IAM role
eksctl create iamserviceaccount \
  --name my-service-account \
  --namespace production \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

### IRSA Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:production:my-service-account",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

---

## Pod Identity vs IRSA

| Feature | EKS Pod Identity | IRSA |
|---------|-----------------|------|
| Setup complexity | Simple | Moderate |
| OIDC provider required | No | Yes |
| Trust policy | AWS-managed | Customer-managed |
| Multi-tenancy | Native support | Requires role per SA |
| Region support | Limited (expanding) | All regions |
| Credential delivery | Pod Identity Agent | OIDC token projection |
| Recommended | Yes (new clusters) | Yes (existing clusters) |

---

## Pod Security Standards

| Level | Description | Use Case |
|-------|-------------|----------|
| Privileged | Unrestricted | System pods, monitoring |
| Baseline | Minimally restrictive | General workloads |
| Restricted | Heavily restricted | Security-sensitive workloads |

### Pod Security Admission

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Secure Pod Specification

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  serviceAccountName: my-service-account
  automountServiceAccountToken: false
  containers:
    - name: app
      image: my-app:latest
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
```

---

## Network Policies

```yaml
# Deny all ingress by default in production namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow specific pod communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - port: 5432
          protocol: TCP
```

---

## Secrets Management

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: production
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded
  password: cGFzc3dvcmQ=  # base64 encoded
```

### External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-external-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-secret
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: prod/myapp/password
        version: AWSCURRENT
```

---

## Security Best Practices

| Area | Best Practice |
|------|---------------|
| IAM | Use least privilege, enable MFA, rotate credentials |
| RBAC | Use namespace-scoped roles, avoid cluster-admin |
| Network | Enable NetworkPolicy, use private endpoints |
| Secrets | Use Pod Identity/IRSA, encrypt at rest |
| Pod Security | Run as non-root, read-only filesystem, drop capabilities |
| Images | Scan for vulnerabilities, use minimal base images |
| Audit | Enable API server logging, use GuardDuty |
| Encryption | Enable KMS encryption for secrets, encrypt volumes |
| Runtime | Use Falco or similar for runtime threat detection |
| Supply Chain | Use image signing, verify digests |

---

## References

- [EKS Security](https://docs.aws.amazon.com/eks/latest/userguide/security.html)
- [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
