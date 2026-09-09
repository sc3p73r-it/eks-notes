# 18. EKS Encryption

## Overview

Encryption protects data at rest and in transit. EKS integrates with AWS KMS for encryption of secrets, volumes (EBS/EFS), container images (ECR), and S3 objects. TLS protects traffic between components.

---

## Encryption Layers

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      EKS Encryption Layers                               │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │               DATA AT REST                                         │ │
│  │                                                                     │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │ │
│  │  │  EKS Secrets │  │    EBS       │  │    EFS       │            │ │
│  │  │  (KMS)       │  │  (encrypted) │  │  (encrypted) │            │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │ │
│  │                                                                     │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │ │
│  │  │     ECR      │  │      S3      │  │Overview Cache│            │ │
│  │  │ (AES256/KMS) │  │ (SSE-S3/KMS) │  │    (KMS)     │            │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                DATA IN TRANSIT (TLS)                               │ │
│  │                                                                     │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │ │
│  │  │  Node ↔  │  │  Pod ↔   │  │  Pod ↔   │  │  LB ↔    │          │ │
│  │  │Control   │  │  Pod     │  │  Service │  │  Internet│          │ │
│  │  │Plane TLS │  │ (mTLS)   │  │  (CNI)   │  │  (ACM)   │          │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## AWS KMS Architecture

### Key Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   KMS Key Hierarchy                                       │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │             KMS Customer Master Key (CMK)                          │ │
│  │             (remains in KMS, never leaves)                         │ │
│  └───────────────────────────┬───────────────────────────────────────┘ │
│                              │ Encrypt/Decrypt (envelope)              │
│  ┌───────────────────────────▼───────────────────────────────────────┐ │
│  │                     Data Encryption Keys (DEK)                     │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │ │
│  │  │   EKS        │  │   EBS        │  │   EFS        │            │ │
│  │  │  Secrets     │  │   Volumes    │  │   Filesystems│            │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### Envelope Encryption Flow

1. Application requests to encrypt data
2. KMS generates a Data Encryption Key (DEK) encrypted by the CMK
3. Application encrypts data with the plaintext DEK
4. Encrypted data and encrypted DEK are stored together
5. During decryption, KMS decrypts the DEK, application decrypts the data

---

## EKS Secrets Encryption

```bash
# Create KMS key
aws kms create-key --description "EKS Secrets Encryption"

# Enable EKS secrets encryption
aws eks update-cluster-config --name my-cluster \
  --encryption-config "{\"resources\":[\"secrets\"],\"provider\":{\"keyArn\":\"arn:aws:kms:us-east-1:123456789012:key/abc-123\"}}"

# Verify encryption config
aws eks describe-cluster --name my-cluster --query "cluster.encryptionConfig"
```

**Note**: Enabling secrets encryption only affects new/updated secrets. Existing Kubernetes Secrets continue as-is (they will be encrypted when next written).

---

## EBS Encryption

```yaml
# Encrypted StorageClass (recommended)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789012:key/ebs-key"
```

```bash
# Set default EBS encryption for account
aws ec2 enable-ebs-encryption-by-default

# Verify
aws ec2 get-ebs-encryption-by-default
```

---

## EFS Encryption

```bash
# Create encrypted EFS filesystem
aws efs create-file-system \
  --creation-token my-efs \
  --performance-mode generalPurpose \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/efs-key \
  --tags Key=Name,Value=encrypted-fs
```

---

## S3 Encryption

```bash
# Create bucket with SSE-S3
aws s3api create-bucket \
  --bucket my-bucket \
  --create-bucket-configuration LocationConstraint=us-east-1 \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Create bucket with SSE-KMS
aws s3api create-bucket --bucket my-bucket-kms \
  --create-bucket-configuration LocationConstraint=us-east-1 \
  --server-side-encryption-configuration '{
    "Rules":[{
      "ApplyServerSideEncryptionByDefault":{
        "SSEAlgorithm":"aws:kms",
        "KMSMasterKeyID":"arn:aws:kms:us-east-1:123456789012:key/s3-key"
      }
    }]
  }'
```

---

## ECR Encryption

```bash
# Create repository with KMS encryption
aws ecr create-repository \
  --repository-name my-app \
  --encryption-configuration \
  encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123456789012:key/ecr-key
```

---

## TLS in Transit

### EKS Control Plane TLS

- All traffic to the API server uses TLS (port 443)
- AWS manages certificates for the control plane
- Certificates are rotated automatically

### Pod-to-Pod mTLS

Use a service mesh for mTLS between pods:

```yaml
# Istio PeerAuthentication
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Load Balancer TLS (ACM)

```bash
# Import or request ACM certificate
aws acm request-certificate \
  --domain-name app.example.com \
  --validation-method DNS
```

---

## KMS Key Management Best Practices

| Practice | Recommendation |
|----------|----------------|
| Key rotation | Enable automatic annual rotation for CMKs |
| Key policies | Restrict to specific principals/services |
| Multi-Region | Use multi-Region keys for DR |
| Auditing | Track key usage in CloudTrail |
| Cost | KMS costs $1/month per CMK + $0.03 per 10k requests |
| Encryption | Prefer KMS encryption over plaintext for compliance |

---

## Compliance Considerations

| Standard | Requirement |
|----------|-------------|
| HIPAA | Encryption at rest required |
| PCI DSS | TLS 1.2+ in transit, encryption at rest |
| SOC 2 | Encryption controls documented and tested |
| FedRAMP | FIPS-validated cryptographic modules |
| GDPR | Data protection, encryption for PII |

---

## Troubleshooting

```bash
# Verify EKS secrets encryption
aws eks describe-cluster --name my-cluster --query "cluster.encryptionConfig"

# Verify EBS volume encryption
aws ec2 describe-volumes --filters Name=volume-id,Values=vol-123 --query "Volumes[0].Encrypted"

# Test KMS permissions
aws kms decrypt --ciphertext-blob fileb://encrypted.bin --key-id arn:aws:kms:...:key/abc-123

# Check KMS key policy
aws kms get-key-policy --key-id arn:aws:kms:...:key/abc-123 --policy-name default
```

---

## References

- [KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html)
- [EKS Secrets Encryption](https://docs.aws.amazon.com/eks/latest/userguide/secrets-encryption.html)
- [EBS Encryption](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSEncryption.html)
- [EFS Encryption](https://docs.aws.amazon.com/efs/latest/ug/encryption.html)
- [ECR Encryption](https://docs.aws.amazon.com/AmazonECR/latest/userguide/encryption-at-rest.html)