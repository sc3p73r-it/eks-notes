# 6. EKS Add-ons

## Overview

EKS add-ons are software packages that extend Kubernetes functionality on EKS. AWS manages the lifecycle of certified add-ons, including installation, configuration, updates, and patching. Add-ons run as DaemonSets or Deployments in the `kube-system` namespace.

---

## EKS-Managed Add-ons

| Addon | Type | Purpose | Default |
|-------|------|---------|---------|
| `vpc-cni` | DaemonSet | Assigns VPC IPs to pods | Yes |
| `kube-proxy` | DaemonSet | Maintains network rules | Yes |
| `coredns` | Deployment | Cluster DNS resolution | Yes |
| `aws-ebs-csi-driver` | DaemonSet | EBS volume lifecycle | No (opt-in) |

### Manage Add-ons via CLI

```bash
# List installed addons
aws eks list-addons --cluster-name my-cluster

# Describe addon
aws eks describe-addon --cluster-name my-cluster --addon-name vpc-cni

# Install EBS CSI driver
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts OVERWRITE

# Update addon
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --resolve-conflicts OVERWRITE \
  --configuration-values '{"enableNetworkPolicy":"true"}'

# Delete addon (keeps resources)
aws eks delete-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver
```

---

## CoreDNS

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    kube-system namespace                      │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                  CoreDNS Deployment                     │   │
│  │  ┌──────────────┐  ┌──────────────┐                   │   │
│  │  │ CoreDNS Pod 1│  │ CoreDNS Pod 2│                   │   │
│  │  │  ┌─────────┐ │  │  ┌─────────┐ │                   │   │
│  │  │  │coredns  │ │  │  │coredns  │ │                   │   │
│  │  │  │:53       │ │  │  │:53       │ │                   │   │
│  │  │  └─────────┘ │  │  └─────────┘ │                   │   │
│  │  └──────────────┘  └──────────────┘                   │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │                  kube-dns Service (ClusterIP)           │   │
│  │  IP: 10.100.0.10                                      │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### Production Recommendations

- Run at least 2 replicas
- Use cluster-proportional-autoscaler for scaling
- Monitor CoreDNS latency and cache hit rate
- Configure custom DNS for internal services

---

## kube-proxy

### Architecture

kube-proxy runs on every node and maintains network rules that implement Kubernetes Service abstraction:

- **iptables mode** (default): Uses Linux netfilter for DNAT and load balancing
- **IPVS mode**: Uses Linux IP Virtual Server for better performance at scale

### When to Use IPVS

- Cluster with 1,000+ Services
- Need consistent latency across all Service endpoints
- High throughput requirements

```bash
# Check kube-proxy mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# Switch to IPVS mode
kubectl set env daemonset kube-proxy -n kube-system mode=ipvs
```

---

## Amazon VPC CNI

### Key Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `WARM_IP_TARGET` | 1 | Number of IPs to keep warm per node |
| `WARM_ENI_TARGET` | 1 | Number of ENIs to keep warm per node |
| `ENABLE_PREFIX_DELEGATION` | false | Use prefix delegation instead of secondary IPs |
| `ENABLE_POD_ENI` | false | Enable pod-level ENI (Security Groups for Pods) |
| `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` | false | Use custom network config |
| `POD_SECURITY_GROUP_ENFORCING_MODE` | off | Pod security group enforcement mode |

```bash
# Enable prefix delegation
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Set warm IP target
kubectl set env daemonset aws-node -n kube-system WARM_IP_TARGET=5

# Enable network policy
kubectl set env daemonset aws-node -n kube-system ENABLE_NETWORK_POLICY=true
```

---

## EBS CSI Driver

### Architecture

```
┌─────────────────────────────────────────────────────────────-┐
│                    kube-system namespace                     │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              EBS CSI Controller (Deployment)          │   │
│  │  ┌──────────────┐  ┌──────────────┐                   │   │
│  │  │  Controller  │  │  Snapshotter │                   │   │
│  │  │  (2 replicas)│  │              │                   │   │
│  │  └──────────────┘  └──────────────┘                   │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              EBS CSI Node (DaemonSet)                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │   │
│  │  │  Node Pod 1  │  │  Node Pod 2  │  │  Node Pod 3  │ │   │
│  │  │  (per node)  │  │  (per node)  │  │  (per node)  │ │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Installation

```bash
# Install as EKS addon
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts OVERWRITE

# Verify
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Retain
```

---

## EFS CSI Driver

### Installation

```bash
# Install via Helm
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=true \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<role-arn>
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-1234567890abcdef0
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
```

---

## AWS Load Balancer Controller

### Installation

```bash
# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Key Features

- Manages ALB (Layer 7) and NLB (Layer 4)
- Supports Ingress and Gateway API
- IP-based target type for pods
- WAF integration
- WAFv2 integration
- Shield integration
- Certificate discovery from ACM

---

## Mountpoint for Amazon S3

```bash
# Install via Helm
helm repo add aws-mountpoint-s3-csi-driver https://awslabs.github.io/mountpoint-s3-csi-driver
helm install mountpoint-s3-csi-driver aws-mountpoint-s3-csi-driver/mountpoint-s3-csi-driver \
  --namespace kube-system
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: s3-existing-bucket
provisioner: s3.csi.aws.com
parameters:
  bucketName: my-bucket
  region: us-east-1
  authenticationSource: pod-identity
```

---

## CloudWatch Observability

```bash
# Install via EKS addon
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability \
  --resolve-conflicts OVERWRITE
```

---

## Upgrade Strategy

1. **Check compatibility**: Verify addon version supports current K8s version
2. **Review changelog**: Check for breaking changes
3. **Update in staging first**: Test on non-production cluster
4. **Update production**: Use `--resolve-conflicts OVERWRITE`
5. **Validate**: Check addon pods, application functionality
6. **Monitor**: Watch for errors in CloudWatch

---

## Troubleshooting

```bash
# Check addon status
aws eks describe-addon --cluster-name my-cluster --addon-name vpc-cni

# Check addon pods
kubectl get pods -n kube-system -l k8s-app=aws-node

# Check addon logs
kubectl logs -n kube-system -l k8s-app=aws-node -c aws-node

# Check addon configuration
kubectl get daemonset aws-node -n kube-system -o yaml

# Check events
kubectl get events -n kube-system --sort-by=.metadata.creationTimestamp
```

---

## References

- [EKS Add-ons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html)
- [CoreDNS](https://docs.aws.amazon.com/eks/latest/best-practices/coredns.html)
- [VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)
- [EFS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)
