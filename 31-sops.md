# 31. Production SOPs

## Overview

Standard Operating Procedures (SOPs) for common EKS operational tasks. Each SOP includes prerequisites, architecture, steps, commands, validation, expected result, rollback, troubleshooting, and security considerations.

---

## SOP 1: Create EKS Cluster

### Prerequisites
- AWS CLI configured with adequate permissions
- VPC with public/private subnets (3 AZs)
- IAM roles (cluster role + node role) created
- AWS account quota available

### Steps

```bash
# 1. Create IAM role for cluster
aws iam create-role --role-name eks-cluster-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"eks.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'
aws iam attach-role-policy --role-name eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# 2. Create cluster (encryption + logging)
aws eks create-cluster \
  --name my-cluster \
  --role-arn arn:aws:iam::123456789012:role/eks-cluster-role \
  --resources-vpc-config \
    subnetIds=subnet-a,subnet-b,subnet-c,securityGroupIds=sg-123 \
    endpointPublicAccess=true,endpointPrivateAccess=true \
  --kubernetes-version 1.31 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# 3. Wait for ACTIVE
aws eks wait cluster-active --name my-cluster

# 4. Update kubeconfig
aws eks update-kubeconfig --name my-cluster --region us-east-1
```

### Validation
```bash
kubectl get svc
aws eks describe-cluster --name my-cluster --query "cluster.status"
```

### Expected Result
Cluster status `ACTIVE`, `kubectl get svc` returns kube-system services.

---

## SOP 2: Configure IAM Access

### Steps

```bash
# Create IAM role for admin access
aws iam create-role --role-name eks-admin --assume-role-policy-document file://trust.json

# Create access entry
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/eks-admin \
  --type STANDARD

# Associate admin policy
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/eks-admin \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

### Validation
```bash
aws eks list-access-entries --cluster-name my-cluster
kubectl auth can-i create deployments -n kube-system --as=$(aws sts get-caller-identity --query Arn --output text)
```

---

## SOP 3: Configure EKS Access Entries (Namespace-scoped)

### Steps

```yaml
# Create ClusterRole for namespace admin
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  - apiGroups: [""]
    resources: ["namespaces","nodes","persistentvolumes"]
    verbs: ["get","list"]
```

```bash
# Associate Namespace-scoped access
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:role/namespace-admin \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSNamespaceAdminPolicy \
  --access-scope type=namespace,namespaces='["production"]'
```

### Validation
```bash
kubectl auth can-i create deployments -n production --as=system:principal:admin-role
```

---

## SOP 4: Configure Managed Node Group

### Steps

```bash
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name production-nodes \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --instance-types m5.large m5.xlarge \
  --scaling-config minSize=3,maxSize=20,desiredSize=6 \
  --subnets subnet-a subnet-b subnet-c \
  --ami-type AL2_x86_64 \
  --disk-size 100
```

### Validation
```bash
kubectl get nodes -l eks.amazonaws.com/nodegroup=production-nodes
```

---

## SOP 5: Install EKS Add-ons

### Steps

```bash
aws eks create-addon --cluster-name my-cluster --addon-name vpc-cni --resolve-conflicts OVERWRITE
aws eks create-addon --cluster-name my-cluster --addon-name kube-proxy
aws eks create-addon --cluster-name my-cluster --addon-name coredns
aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver
```

### Validation
```bash
aws eks list-addons --cluster-name my-cluster
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

---

## SOP 6: Configure AWS Load Balancer Controller

### Steps

```bash
# Create IAM policy
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

# Create service account
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Validation
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

---

## SOP 7: Configure EBS CSI

### Steps

```bash
# Enable addon
aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver
# Verify storageclass
kubectl get sc
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  encrypted: "true"
EOF
```

### Validation
```bash
kubectl get sc gp3
```

---

## SOP 8: Configure EFS CSI

### Steps

```bash
# Create file system + mount targets
aws efs create-file-system --creation-token eks-efs --performance-mode generalPurpose --encrypted
aws efs create-mount-target --file-system-id fs-xxx --subnet-id subnet-a --security-groups sg-xxx
aws efs create-mount-target --file-system-id fs-xxx --subnet-id subnet-b --security-groups sg-xxx
aws efs create-mount-target --file-system-id fs-xxx --subnet-id subnet-c --security-groups sg-xxx

# Install driver
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver -n kube-system
```

### Validation
```bash
kubectl get pods -n kube-system | grep efs-csi
```

---

## SOP 9: Configure Pod Identity

### Steps

```bash
# Create role
aws iam create-role --role-name my-pod-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}'
aws iam attach-role-policy --role-name my-pod-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create association
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace production \
  --service-account my-sa \
  --role-arn arn:aws:iam::123456789012:role/my-pod-role
```

### Validation
```bash
aws eks list-pod-identity-associations --cluster-name my-cluster
kubectl -n production get sa my-sa -o yaml
kubectl exec -it <pod> -- aws sts get-caller-identity
```

---

## SOP 10: Configure Secrets Manager Integration

### Steps

```bash
# Store secret
aws secretsmanager create-secret --name prod/myapp/database \
  --secret-string '{"password":"P@ssword123!"}'

# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm upgrade -i external-secrets external-secrets/external-secrets -n external-secrets --create-namespace

# Create SecretStore + ExternalSecret (see 17-secrets-management.md)
kubectl apply -f secret-store.yaml
kubectl apply -f external-secret.yaml
```

### Validation
```bash
kubectl get externalsecret -n production
kubectl get secret app-secrets -n production
```

---

## SOP 11: Deploy Application

### Prerequisites
- ECR image pushed
- GitOps repo / manifests prepared

### Steps

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl rollout status deployment/my-app -n production
```

### Validation
```bash
kubectl get pods -n production
kubectl get ingress -n production
```

### Rollback
```bash
kubectl rollout undo deployment/my-app -n production
```

---

## SOP 12: Configure Autoscaling

### Steps

```bash
cat <<EOF | kubectl apply -f -
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
EOF
```

### Validation
```bash
kubectl get hpa -n production
kubectl describe hpa my-app-hpa -n production
```

---

## SOP 13: Configure Monitoring

### Steps

```bash
# Container Insights
curl https://raw.githubusercontent.com/aws-observability/aws-observability-accelerator/main/artifacts/container-insights/fluent-bit.yaml | \
  sed "s/your-cluster-name/my-cluster/g; s/your-cluster-region/us-east-1/g" | kubectl apply -f -

# Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade -i prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

### Validation
```bash
kubectl get pods -n monitoring
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

---

## SOP 14: Configure Logging

### Steps

```bash
# Enable control plane logging
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Deploy Fluent Bit (see 12-logging-architecture.md)
kubectl apply -f fluent-bit.yaml
```

### Validation
```bash
aws logs describe-log-groups --log-group-name-prefix /aws/eks/my-cluster
aws logs filter-log-events --log-group-name /aws/eks/my-cluster/cluster --limit 10
```

---

## SOP 15: Upgrade EKS

### Steps

```bash
# 1. Backup
velero backup create pre-upgrade --ttl 168h

# 2. Upgrade control plane
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.32
aws eks wait cluster-active --name my-cluster

# 3. Upgrade addons
for addon in vpc-cni kube-proxy coredns; do
  aws eks update-addon --cluster-name my-cluster --addon-name $addon --resolve-conflicts OVERWRITE
done

# 4. Upgrade node group
aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name production-nodes
```

### Validation
```bash
kubectl get nodes -l eks.amazonaws.com/nodegroup=production-nodes -o wide
kubectl get pods -A | grep -v Running
```

---

## SOP 16: Troubleshoot Nodes

### Steps

```bash
kubectl get nodes -o wide
kubectl describe node <node-name> | grep -A10 "Conditions:"
kubectl top node
kubectl get events --field-selector involvedObject.kind=Node --sort-by=.metadata.creationTimestamp
```

For NotReady: SSH/SSM to node, check `systemctl status kubelet`, `journalctl -u kubelet`.

---

## SOP 17: Troubleshoot Pods

### Steps

```bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --tail=100
kubectl logs <pod> -n <namespace> --previous --tail=100
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

---

## SOP 18: Troubleshoot Networking

### Steps

```bash
# DNS
kubectl run dnstest --image=busybox --rm -it -- nslookup kubernetes.default

# Pod connectivity
kubectl exec -it <pod> -- curl http://<target>

# CNI
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl logs -n kube-system -l k8s-app=aws-node -c aws-node

# LB
kubectl describe svc <svc>
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

---

## SOP 19: Backup Cluster

### Steps

```bash
# Velero
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.10.0 --bucket eks-backups \
  --backup-location-config region=us-east-1 --no-secret

# Backup
velero backup create full-backup --include-namespaces production --ttl 168h
velero backup describe full-backup

# EBS snapshot (see 07-storage-deep-dive.md + 20-disaster-recovery.md)
```

---

## SOP 20: Disaster Recovery

### Steps

```bash
# 1. Restore from Velero backup
velero restore create --from-backup full-backup

# 2. Restore EBS volumes / RDS from snapshots
aws rds restore-db-instance-from-db-snapshot --db-instance-identifier my-db --db-snapshot-identifier my-snapshot

# 3. Update GitOps (Argo CD) to point to DR cluster
kubectl config use-context dr-cluster
argocd app sync my-app

# 4. Switch DNS (Route 53 failover)
# 5. Verify application health
curl -H "Host: app.example.com" https://<dr-alb-dns>/healthz
```

---

## SOP Security Considerations

| SOP | Security note |
|-----|---------------|
| Cluster create | Restrict public access CIDRs, enable private endpoint |
| IAM access | Least privilege, temporary credentials |
| Node groups | Disable SSH publicly, use SSM |
| Add-ons | Only install required addons, pin versions |
| Pod Identity | Grant specific roles, not admin |
| Secrets | Never log secret values |
| Backups | Encrypt backups, control backup access |
| DR | Keep DR cluster private, credentials secured |

---

## References

- [EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html)
- [Kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Velero](https://velero.io/docs/)