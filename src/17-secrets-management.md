# 17. EKS Secrets Management

## Overview

Secrets management in EKS determines how sensitive data (passwords, API keys, certificates) is stored, accessed, rotated, and protected. The options range from native Kubernetes Secrets to AWS Secrets Manager and external tools like External Secrets Operator or the Secrets Store CSI Driver.

---

## Approach Comparison

| Feature | Kubernetes Secrets | Secrets Manager | Parameter Store | Secrets Store CSI | External Secrets Operator |
|---------|-------------------|-----------------|-----------------|-------------------|---------------------------|
| Storage | etcd (encrypted) | AWS-managed | AWS-managed | External, mounted | Syncs external → K8s Secret |
| Encryption | KMS envelope | KMS | KMS | KMS | KMS |
| Rotation | Manual | Automatic | Manual | Manual | Manual |
| Access | RBAC | IAM | IAM | IAM + RBAC | IAM |
| Kubernetes CRD | Native | No | No | Yes | Yes |
| Multi-tenant | Namespace | IAM | IAM | IAM + Namespace | IAM + Namespace |
| Cost | Free | $0.40/secret/month | $0.05/param/month | Free | Free |

---

## Kubernetes Secrets

### Storage

Kubernetes Secrets are stored in etcd. With EKS, etcd is encrypted at rest via KMS envelope encryption.

### Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=  # base64 encoded
  API_KEY: c2VjcmV0LWtleQ==
stringData:
  # plain text, encoded at creation
  CONFIG_VALUE: "my-config"
```

```bash
# Create from literal
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD='password123' \
  --from-literal=API_KEY='sk-abc123' \
  --namespace production

# Create from file
kubectl create secret generic app-secrets \
  --from-file=./secrets/env.json

# View
kubectl get secrets -n production
kubectl get secret app-secrets -n production -o yaml
```

### Limitations

- Data is base64-encoded, NOT encrypted
- Difficult to rotate (requires redeploy)
- Can leak in logs if not careful
- No built-in multi-tenancy at secret level

---

## AWS Secrets Manager

### IAM Policy for Secrets Manager

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/*"
      ]
    }
  ]
}
```

### Store a Secret

```bash
aws secretsmanager create-secret \
  --name prod/myapp/database \
  --secret-string '{"username":"admin","password":"S3cur3Passw0rd!!"}' \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc-123

# Update secret value
aws secretsmanager put-secret-value \
  --secret-id prod/myapp/database \
  --secret-string '{"username":"admin","password":"newpass"}'

# Automatic rotation (with Lambda)
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/database \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:rotate-db \
  --rotation-rules AutomaticallyAfterDays=30
```

### Application Access

Applications access Secrets Manager via an IAM credential (IRSA or Pod Identity). Example with boto3/Python:

```python
import boto3
import json
from botocore.config import Config

def get_secret(secret_id, region="us-east-1"):
    client = boto3.client(
        "secretsmanager",
        region_name=region,
        config=Config(connect_timeout=5, read_timeout=5)
    )
    resp = client.get_secret_value(SecretId=secret_id)
    return json.loads(resp["SecretString"])
```

---

## AWS Systems Manager Parameter Store

### Store a Parameter

```bash
aws ssm put-parameter \
  --name "/prod/myapp/config/database-url" \
  --value "postgres://user:pass@host:5432/db" \
  --type SecureString \
  --key-id alias/aws/ssm \
  --tier Advanced

# Get parameter
aws ssm get-parameter --name "/prod/myapp/config/database-url" --with-decryption

# List parameters
aws ssm describe-parameters --parameter-filters Key=Name,Option=BeginsWith,Values=/prod
```

---

## Secrets Store CSI Driver

The Secrets Store CSI Driver mounts secrets as volumes or environment variables at pod startup.

### Install

```bash
# Install via Helm
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install secrets-store-csi-driver secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system
```

### AWS Provider

```bash
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

### SecretProviderClass

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/myapp/database"
        objectType: "secretsmanager"
      - objectName: "PROD_CONFIGURED"
        objectAlias: "config-value"
        objectType: "ssmparameter"
```

### Pod Usage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: production
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "app-secrets"
```

---

## External Secrets Operator

External Secrets Operator synchronizes secrets from external systems into Kubernetes Secrets.

### Install

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

### SecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        # Uses Pod Identity / IRSA of the controller
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

### ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-secrets-k8s
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/myapp/database
        property: password
    - secretKey: API_KEY
      remoteRef:
        key: prod/myapp/api
        property: api_key
```

### Rotating ExternalSecret

Kubernetes automatically creates the associated K8s Secret. Applications referencing the Secret by name use the updated value upon restart. To force pods to pick up changes without restart, use reloader or similar tooling.

---

## Recommended Enterprise Approach

```
┌─────────────────────────────────────────────────────────────────┐
│               Enterprise Secrets Architecture                     │
│                                                                   │
│  ┌────────────────────────────┐  ┌────────────────────────────┐ │
│  │  AWS Secrets Manager        │  │  SSM Parameter Store       │ │
│  │  - Application secrets     │  │  - Config parameters       │ │
│  │  - Rotation-enabled        │  │  - SecureString tier       │ │
│  └─────────────┬──────────────┘  └─────────────┬──────────────┘ │
│                │                               │                 │
│                ▼                               ▼                 │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │          External Secrets Operator (or CSI Driver)           ││
│  └──────────────────────────────┬───────────────────────────────┘│
│                                 │                                 │
│                                 ▼                                 │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │              Kubernetes Secrets (namespaced)                ││
│  │              (ephemeral, synced, encrypted at rest)          ││
│  └──────────────────────────────┬───────────────────────────────┘│
│                                 │                                 │
│                                 ▼                                 │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │                        Pods / Apps                           ││
│  └──────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Recommended Approach

1. **Store secrets centrally** in AWS Secrets Manager (with automatic rotation where possible)
2. **Sync via External Secrets Operator** for dynamic, auto-refreshed Kubernetes Secrets
3. **Use Secrets Store CSI Driver** for files that must be mounted without being visible in Kubernetes
4. **Never store plaintext secrets** in Git repositories or image tags
5. **Rotate secrets** at least every 90 days (or on employee departure / suspected leak)
6. **Track access** via CloudTrail / CloudWatch for secrets read events

---

## Security Best Practices

| Practice | Recommendation |
|----------|----------------|
| Storage | Prefer AWS-managed (Secrets Manager/SSM) over plain K8s Secrets |
| Encryption | Always enable KMS encryption |
| Rotation | Enable automatic rotation for databases, API keys |
| IAM | Grant least-privilege access to individual secrets |
| RBAC | Restrict Secret access by namespace |
| Auditing | Monitor GetSecretValue API calls in CloudTrail |
| Git | Use secret scanners (GitGuardian, Gitleaks) in CI |
| App access | Prefer IRSA/Pod Identity over static credentials |

---

## Troubleshooting

```bash
# Check ExternalSecret status
kubectl get externalsecrets -n production
kubectl describe externalsecret app-secrets -n production

# Check CSI driver provider
kubectl get pods -n kube-system | grep secrets-store
kubectl logs -n kube-system -l app=secrets-store-csi-driver

# Test secret access from pod
kubectl exec -it app-pod -- cat /mnt/secrets/config-value

# Verify IAM permissions
kubectl describe pod app-pod -n production | grep -i "serviceaccount"
aws secretsmanager describe-secret --secret-id prod/myapp/database
```

---

## References

- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/)
- [External Secrets Operator](https://external-secrets.io/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)