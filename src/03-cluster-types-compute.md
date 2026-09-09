# 3. EKS Cluster Types and Compute Options

## Overview

Amazon EKS provides four primary compute models: Managed Node Groups, Self-Managed Nodes, EKS Auto Mode, and AWS Fargate. Each model differs in management overhead, customization, cost, and use case. The choice depends on workload requirements, team expertise, and operational preferences.

---

## Comparison Matrix

| Feature | Managed Node Groups | Self-Managed Nodes | EKS Auto Mode | Fargate |
|---------|--------------------|--------------------|---------------|---------|
| **Management** | AWS manages ASG, AMI | Customer manages everything | AWS manages everything | AWS manages infra |
| **AMI** | EKS-optimized AMI | Customer-provided | AWS-managed | AWS-managed |
| **Scaling** | ASG-based | Customer-configured | Karpenter-based | Pod-based |
| **Instance selection** | Customer chooses | Customer chooses | AWS optimizes | Limited (1-16 vCPU) |
| **Customization** | Launch templates | Full control | Limited | None |
| **Cost model** | EC2 pricing | EC2 pricing | EC2 pricing | Per-pod vCPU/memory |
| **Spot support** | Yes | Yes | Yes | Yes |
| **GPU support** | Yes | Yes | Yes | No |
| **Pod-level IAM** | IRSA / Pod Identity | IRSA / Pod Identity | IRSA / Pod Identity | IRSA / Pod Identity |
| **Storage** | EBS, EFS | EBS, EFS | EBS (auto-managed) | EFS only |
| **Best for** | General workloads | Custom requirements | Simplified operations | Stateless, batch |

---

## Managed Node Groups

Managed Node Groups create and manage EC2 instances using AWS Auto Scaling Groups (ASGs). AWS handles AMI updates, node lifecycle, and scaling configuration.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                  EKS Control Plane                    │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────┴─────────────────────────────┐
│              Customer VPC                             │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │           Auto Scaling Group                     │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐         │ │
│  │  │ Node 1  │  │ Node 2  │  │ Node 3  │         │ │
│  │  │ m5.large│  │ m5.large│  │ m5.large│         │ │
│  │  │         │  │         │  │         │         │ │
│  │  │┌──────┐ │  │┌──────┐ │  │┌──────┐ │         │ │
│  │  ││Pod A │ │  ││Pod C │ │  ││Pod E │ │         │ │
│  │  ││Pod B │ │  ││Pod D │ │  ││Pod F │ │         │ │
│  │  │└──────┘ │  │└──────┘ │  │└──────┘ │         │ │
│  │  └─────────┘  └─────────┘  └─────────┘         │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

### Lifecycle Management

```bash
# Create managed node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --instance-types m5.large m5.xlarge \
  --scaling-config minSize=2,maxSize=10,desiredSize=3 \
  --subnets subnet-0abc123 subnet-0def456 \
  --ami-type AL2_x86_64 \
  --disk-size 50 \
  --labels environment=production,team=platform \
  --taints key=dedicated,value=production,effect=NO_SCHEDULE

# Update scaling configuration
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes \
  --scaling-config minSize=3,maxSize=20,desiredSize=5

# Delete node group
aws eks delete-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes
```

### AMI Types

| AMI Type | Description | Use Case |
|----------|-------------|----------|
| `AL2_x86_64` | Amazon Linux 2, x86_64 | General workloads |
| `AL2_ARM_64` | Amazon Linux 2, ARM64 (Graviton) | Cost-optimized workloads |
| `AL2_x86_64_GPU` | Amazon Linux 2, GPU-optimized | ML/AI, rendering |
| `BOTTLEROCKET_x86_64` | Bottlerocket, x86_64 | Container-optimized |
| `BOTTLEROCKET_ARM_64` | Bottlerocket, ARM64 | Container-optimized, cost |
| `CUSTOM` | Customer-provided AMI | Custom requirements |

### Launch Templates

```yaml
# Launch template for custom node configuration
apiVersion: ec2.launchtemplate/v1
LaunchTemplate:
  LaunchTemplateName: eks-custom-node
  LaunchTemplateData:
    ImageId: ami-0123456789abcdef0
    InstanceType: m5.large
    UserData: |
      #!/bin/bash
      /etc/eks/bootstrap.sh my-cluster --kubelet-extra-args '--node-labels=env=prod'
    BlockDeviceMappings:
      - DeviceName: /dev/xvda
        Ebs:
          VolumeSize: 100
          VolumeType: gp3
          Iops: 3000
          Throughput: 125
          Encrypted: true
    TagSpecifications:
      - ResourceType: instance
        Tags:
          - Key: Environment
            Value: production
```

### Node Labels and Taints

```bash
# Add labels to existing node
kubectl label nodes ip-10-0-32-10.ec2.internal environment=production
kubectl label nodes ip-10-0-32-10.ec2.internal node-type=gpu

# Add taints
kubectl taint nodes ip-10-0-32-10.ec2.internal dedicated=gpu:NoSchedule

# List labels
kubectl get nodes --show-labels

# List taints
kubectl describe node ip-10-0-32-10.ec2.internal | grep -A5 Taints

# Verify pod scheduling with taints
kubectl get pods -o wide | grep -v node-name
```

### Spot Instances

```bash
# Create Spot node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name spot-nodes \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --instance-types m5.large m5.xlarge m5.2xlarge c5.large c5.xlarge \
  --capacity-type SPOT \
  --scaling-config minSize=0,maxSize=20,desiredSize=3
```

**Spot Interruption Handling:**

```yaml
# AWS Node Termination Handler DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: aws-node-termination-handler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: aws-node-termination-handler
  template:
    metadata:
      labels:
        name: aws-node-termination-handler
    spec:
      serviceAccountName: aws-node-termination-handler
      terminationGracePeriodSeconds: 0
      containers:
        - name: aws-node-termination-handler
          image: public.ecr.aws/bitnami/aws-node-termination-handler:latest
          env:
            - name: NODE_IRSA_ENABLED
              value: "true"
            - name: POD_TERMINATION_GRACE_PERIOD
              value: "30"
      tolerations:
        - operator: Exists
```

---

## Self-Managed Nodes

Self-managed nodes give customers full control over the EC2 instances, AMI, bootstrap script, and lifecycle. This is used when the standard EKS-optimized AMI or managed node group configuration is insufficient.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                  EKS Control Plane                    │
└───────────────────────┬─────────────────────────────┘
                        │
┌───────────────────────┴─────────────────────────────┐
│              Customer VPC                             │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │  Self-Managed Node (Customer ASG/EC2)            │ │
│  │  ┌────────────────────────────────────────────┐ │ │
│  │  │  Custom AMI + Bootstrap Script              │ │ │
│  │  │  ┌─────────┐  ┌─────────┐                  │ │ │
│  │  │  │ kubelet  │  │kube-proxy│                 │ │ │
│  │  │  └─────────┘  └─────────┘                  │ │ │
│  │  │  ┌─────────┐  ┌─────────┐                  │ │ │
│  │  │  │containerd│  │  Pods   │                  │ │ │
│  │  │  └─────────┘  └─────────┘                  │ │ │
│  │  └────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

### Bootstrap Process

```bash
# Amazon Linux 2 bootstrap
/etc/eks/bootstrap.sh <cluster-name> \
  --kubelet-extra-args '--node-labels=node-role.kubernetes.io/worker= --register-with-taints=node-role.kubernetes.io/worker=:NoSchedule' \
  --b64-cluster-ca <ca-cert> \
  --apiserver-endpoint <api-endpoint>

# Bottlerocket bootstrap (via user data TOML)
[settings.kubernetes]
cluster-name = "my-cluster"
api-server = "https://XXXXX.eks.us-east-1.amazonaws.com"
cluster-certificate = "/etc/kubernetes/pki/ca.crt"
[settings.kubernetes.node-labels]
environment = "production"
[settings.kubernetes.node-taints]
"dedicated=gpu:NoSchedule" = true
```

### Upgrade Considerations

Self-managed nodes require the customer to:

1. Build or obtain updated AMIs
2. Update launch templates with new AMI IDs
3. Perform rolling replacement of nodes
4. Validate workloads on new nodes
5. Remove old nodes

```bash
# Describe launch template versions
aws ec2 describe-launch-template-versions --launch-template-id lt-0123456789abcdef0

# Create new launch template version with updated AMI
aws ec2 create-launch-template-version \
  --launch-template-id lt-0123456789abcdef0 \
  --source-version 1 \
  --launch-template-data '{"ImageId":"ami-new123456"}'

# Update auto scaling group
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name eks-self-managed \
  --launch-template "LaunchTemplateId=lt-0123456789abcdef0,Version=2"
```

---

## EKS Auto Mode

EKS Auto Mode is a fully managed compute option where AWS provisions, manages, and scales nodes automatically. It extends the EKS control plane to manage the data plane, including compute, networking, and storage.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    EKS Control Plane                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Auto Mode Controller                                      │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │   │
│  │  │ Compute  │  │Networking│  │ Storage  │               │   │
│  │  │ Manager  │  │ Manager  │  │ Manager  │               │   │
│  │  └──────────┘  └──────────┘  └──────────┘               │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────────┐
│                    Customer VPC                                   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Auto-Managed Nodes                                       │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                   │   │
│  │  │ Node 1  │  │ Node 2  │  │ Node 3  │  (auto-provisioned)│  │
│  │  │┌──────┐ │  │┌──────┐ │  │┌──────┐ │                   │   │
│  │  ││ Pods │ │  ││ Pods │ │  ││ Pods │ │                   │   │
│  │  │└──────┘ │  │└──────┘ │  │└──────┘ │                   │   │
│  │  └─────────┘  └─────────┘  └─────────┘                   │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Auto-Managed Storage                                     │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  EBS Volumes (auto-provisioned via gp3 StorageClass)│ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └───────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

### How Compute Provisioning Works

1. Pod is scheduled by the Kubernetes scheduler
2. Auto Mode controller detects pending pods
3. Controller selects optimal instance type based on pod requirements
4. Controller provisions EC2 instance in appropriate subnet/AZ
5. Kubelet on new node registers with the API server
6. Pod is placed on the new node

### Enable Auto Mode

```bash
# Enable via CLI
aws eks update-cluster-config --name my-cluster \
  --compute-config '{"enabled":true}' \
  --storage-config '{"blockStorage":{"enabled":true}}' \
  --kubernetes-network-config '{"serviceIpv4Cidr":"172.20.0.0/16"}'

# Verify Auto Mode status
aws eks describe-cluster --name my-cluster \
  --query "cluster.computeConfig"
```

### Auto Mode vs Managed Node Groups

| Aspect | EKS Auto Mode | Managed Node Groups |
|--------|---------------|---------------------|
| Node provisioning | Fully automatic | Customer-configured |
| Instance selection | AWS optimizes | Customer selects |
| Scaling | Automatic based on pods | ASG min/max/desired |
| Networking | Auto-configured | Customer configures CNI |
| Storage | Auto-provisioned EBS | Customer configures CSI |
| Customization | Limited | Launch templates, custom AMI |
| Cost visibility | Less granular | Full EC2 cost visibility |
| Node pools | Not applicable | Named node groups |

### Limitations

- Limited instance type selection (AWS optimizes automatically)
- Cannot use custom AMIs
- Cannot use launch templates
- Cannot attach custom tags to instances
- Limited control over instance placement
- Not suitable for workloads requiring specific hardware (e.g., specific GPU types)
- EBS volumes only (gp3); no EFS auto-provisioning

### When to Use Auto Mode

- **Use Auto Mode** when: simplified operations, general workloads, no custom AMI requirement, no specific instance type requirement
- **Use Managed Node Groups** when: custom AMIs, specific instance types, launch template customization, GPU workloads, node-level taints/labels

---

## EKS Capabilities (Managed Platform Components)

Announced November 2025, **Amazon EKS Capabilities** offload common platform components (ACK, Argo CD, kro) so they run as **fully managed control-plane components on AWS-owned infrastructure** — not on your worker nodes.

| Capability | What it is | Managed by |
|------------|------------|------------|
| **ACK** (AWS Controllers for Kubernetes) | Provision AWS resources (DynamoDB, S3, RDS…) via Kubernetes custom resources | AWS |
| **Argo CD** | GitOps continuous delivery (cluster as deployment target) | AWS |
| **kro** (Kube Resource Orchestrator) | Compose multiple K8s resources into one via `ResourceGraphDefinition` | AWS |

### How It Differs from Self-Install

| Aspect | Traditional self-install | EKS Capabilities |
|--------|--------------------------|-------------------|
| Control plane component | Runs as Deployments on your nodes | Runs on AWS-owned infrastructure |
| Install | Helm install + controller Deployment + pod-level IRSA | Enabled; capability assumes its own IAM role |
| Scaling/patching/upgrades | Customer responsibility | AWS responsibility |
| In-cluster artifacts | Controllers, Deployments, webhooks | Only the CRDs (and any managed namespace) |

### When to Use

- Use **EKS Capabilities** to eliminate platform-tool maintenance (ACK, Argo CD, kro) for production GitOps and AWS resource provisioning.
- Use self-install when you need custom controller settings, pinned versions, or non-AWS-capability tooling (e.g., Flux, Crossplane).

---

## AWS Fargate

Fargate is a serverless compute engine for containers that works with both EKS and ECS. With Fargate, there are no nodes to manage; each pod runs in its own isolated microVM.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    EKS Control Plane                              │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────────┐
│                    AWS Fargate                                     │
│                                                                   │
│  ┌──────────────────────┐  ┌──────────────────────┐             │
│  │  Fargate Profile 1   │  │  Fargate Profile 2   │             │
│  │  (kube-system)       │  │  (production)         │             │
│  │                      │  │                       │             │
│  │  ┌────────────────┐  │  │  ┌────────────────┐  │             │
│  │  │  CoreDNS Pod   │  │  │  │  App Pod       │  │             │
│  │  │  (microVM)     │  │  │  │  (microVM)     │  │             │
│  │  └────────────────┘  │  │  └────────────────┘  │             │
│  │  ┌────────────────┐  │  │  ┌────────────────┐  │             │
│  │  │  kube-proxy    │  │  │  │  Worker Pod    │  │             │
│  │  │  (microVM)     │  │  │  │  (microVM)     │  │             │
│  │  └────────────────┘  │  │  └────────────────┘  │             │
│  └──────────────────────┘  └──────────────────────┘             │
└───────────────────────────────────────────────────────────────────┘
```

### Pod Execution Model

- Each pod runs in an isolated microVM (Firecracker)
- No shared kernel, no shared filesystem between pods
- Each pod gets its own ENI with VPC IP address
- Pods are ephemeral; no local storage persistence

### Fargate Profiles

```bash
# Create Fargate profile
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name production \
  --subnets subnet-0abc123 subnet-0def456 \
  --selectors namespace=production,labels={env=production}

# Create Fargate profile for kube-system
aws eks create-fargate-profile \
  --cluster-name my-cluster \
  --fargate-profile-name kube-system \
  --subnets subnet-0abc123 subnet-0def456 \
  --selectors namespace=kube-system,labels={k8s-app=kube-dns}
```

### Fargate Pod Spec

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
          resources:
            requests:
              cpu: "512m"
              memory: "1Gi"
            limits:
              cpu: "1"
              memory: "2Gi"
      nodeSelector:
        kubernetes.io/os: linux
      tolerations:
        - key: "eks.amazonaws.com/compute-type"
          operator: "Equal"
          value: "fargate"
          effect: "NoSchedule"
```

### Fargate Limitations

| Limitation | Impact |
|------------|--------|
| No DaemonSet support | Cannot run DaemonSets (kube-proxy, VPC CNI, etc.) |
| No privileged containers | Cannot run containers with elevated privileges |
| No GPU support | Cannot use GPU instances |
| Limited resource sizes | 1 vCPU / 2 GB to 16 vCPU / 120 GB |
| Ephemeral storage | Only 20 GB ephemeral storage |
| No instance store | Cannot use NVMe/instance store volumes |
| Slower startup | Cold start: 30-60 seconds |
| No SSH access | Cannot SSH into Fargate tasks |
| Pod-only networking | Each pod gets its own ENI (no shared node network) |
| No host networking | Cannot use hostNetwork: true |

### Fargate Cost Model

Fargate pricing is based on vCPU-hours and GB-hours:

| Resource | Price (us-east-1) |
|----------|-------------------|
| vCPU | $0.04048/hour |
| Memory | $0.004445/GB/hour |
| Storage | $0.000111/GB/hour |

**Cost comparison example (m5.large = 2 vCPU, 8 GB):**

| Option | Monthly Cost |
|--------|-------------|
| EC2 On-Demand (m5.large) | ~$70 |
| EC2 Spot (m5.large) | ~$21 |
| Fargate (2 vCPU, 8 GB) | ~$80 |

### Appropriate Workloads for Fargate

- Stateless web applications
- Batch processing jobs
- CI/CD workloads (Jenkins agents, GitLab runners)
- Scheduled tasks (CronJobs)
- Development/staging environments
- Workloads with unpredictable traffic

### Inappropriate Workloads for Fargate

- DaemonSets (monitoring agents, log collectors)
- Workloads requiring GPU
- Workloads requiring host networking
- StatefulSets with persistent volumes (EFS only, no EBS)
- Workloads requiring SSH access
- Low-latency workloads (cold start latency)

---

## Graviton (ARM64) Instances

Graviton processors offer up to 40% better price-performance compared to x86 instances.

```bash
# Create ARM64 node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name arm-nodes \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --instance-types m6g.large m6g.xlarge \
  --ami-type AL2_ARM_64 \
  --scaling-config minSize=2,maxSize=10,desiredSize=3
```

**Multi-arch considerations:**

- Use multi-arch container images (build for both amd64 and arm64)
- Test applications on ARM64 before production
- Some software may not have ARM64 builds

---

## References

- [EKS Compute Overview](https://docs.aws.amazon.com/eks/latest/userguide/eks-compute.html)
- [Managed Node Groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)
- [Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)
- [EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [EKS Capabilities announcement](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-eks-capabilities/)
- [Karpenter](https://karpenter.sh/docs/)
- [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler)
- [Graviton with EKS](https://github.com/aws/aws-graviton-getting-started)
