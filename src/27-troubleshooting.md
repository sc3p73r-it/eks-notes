# 27. EKS Troubleshooting

## Overview

This is a practical troubleshooting guide for EKS covering cluster, node, pod, networking, IAM, and storage issues. Commands use `kubectl`, `aws eks`, `aws ec2`, `aws iam`, `aws logs`, and `helm`.

---

## Cluster Issues

### Cluster Unavailable

```bash
# Describe cluster
aws eks describe-cluster --name my-cluster --region us-east-1

# Check cluster status
aws eks describe-cluster --name my-cluster --query "cluster.status"

# Check API server endpoint
aws eks describe-cluster --name my-cluster --query "cluster.endpoint"
```

### API Server Issues

```bash
# Check kubeconfig
kubectl config view --minify

# Test API server connectivity
kubectl cluster-info
kubectl get --raw /healthz

# Check API server logs (control plane)
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/cluster \
  --filter-pattern "api server"
```

### Authentication Failures

```bash
# Check IAM credentials
aws sts get-caller-identity

# Update kubeconfig (refresh auth)
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Check access entry
aws eks list-access-entries --cluster-name my-cluster --region us-east-1

# Check aws-auth (legacy)
kubectl get configmap aws-auth -n kube-system -o yaml

# Test permission
kubectl auth can-i get pods --as=<user> -n <namespace>
```

---

## Node Issues

### Node NotReady

```bash
# List nodes
kubectl get nodes -o wide

# Describe node for conditions
kubectl describe node <node-name>

# Check kubelet status (requires SSH/SSM to node)
sudo systemctl status kubelet
sudo journalctl -u kubelet --no-pager | tail -50

# Check node conditions summary
kubectl get node <node-name> \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status}\n{end}'
```

### DiskPressure / MemoryPressure / NetworkUnavailable

```bash
kubectl describe node <node-name> | grep -A10 "Conditions:"

# Check disk usage (on node)
df -h /
sudo journalctl -u kubelet --no-pager | grep -i "pressure\|evict"

# Check network interface (on node)
ip addr show
sudo ethtool eth0
```

### Node Capacity

```bash
# Node capacity/allocatable
kubectl describe node <node-name> | grep -A10 "Capacity:"
kubectl get node <node-name> -o jsonpath='{.status.capacity}'

# Pod resource usage
kubectl top node
```

---

## Pod Issues

### Pending (Scheduling Failure)

```bash
# Describe pod (look for FailedScheduling events)
kubectl describe pod <pod-name> -n <namespace>

# Check node affinity/taints
kubectl get nodes --show-labels
kubectl describe node <node-name> | grep -A5 "Taints:"

# Check resource quotas
kubectl get resourcequota -n <namespace>

# Check if autoscaler can add nodes
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
```

### CrashLoopBackOff

```bash
# Describe pod
kubectl describe pod <pod-name> -n <namespace>

# Check logs
kubectl logs <pod-name> -n <namespace> --tail=50
kubectl logs <pod-name> -n <namespace> --previous --tail=50

# Check previous container exit code
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A3 "lastState"
```

### ImagePullBackOff

```bash
# Describe pod (check for ImagePullBackOff event)
kubectl describe pod <pod-name> -n <namespace>

# Check image reference
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[].image}'

# Test image pull from node (if SSH access)
sudo crictl pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1

# Check node ECR permissions
aws sts get-caller-identity
kubectl get secrets -n <namespace> | grep -i "docker\|ecr"
```

### OOMKilled

```bash
# Describe pod (check for OOMKilling)
kubectl describe pod <pod-name> -n <namespace> | grep -i "killed\|OOM"

# Check container memory limits
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[].resources}'

# Increase memory limit or tune application
kubectl edit deployment <deployment-name> -n <namespace>
```

### ContainerCreating

```bash
# Describe pod
kubectl describe pod <pod-name> -n <namespace> | grep -A5 Events

# Check images
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.initContainers[].image}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[].image}'

# Check volume mounts
kubectl get pod <pod-name> -n <namespace> -o yaml | grep -A5 volumeMounts
```

---

## Networking Issues

### DNS Failure

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS Service
kubectl get svc -n kube-system kube-dns

# Test DNS from a pod
kubectl run dnstest --image=busybox --rm -it -- nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

### Pod Cannot Access Internet

```bash
# Check NAT gateway
aws ec2 describe-nat-gateways --filter Name=state,Values=available

# Check route tables (private subnets must route to NAT)
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>

# Test egress from pod
kubectl exec -it <pod> -- ping 8.8.8.8
kubectl exec -it <pod> -- curl -v https://google.com

# Verify NodePermissions
kubectl describe pod <pod> | grep -i "i-am\|role"
```

### Pod Cannot Communicate with Another Pod

```bash
# Test pod-to-pod
kubectl exec -it <pod1> -- curl http://<pod2-ip>:<port>

# Check security group rules
aws ec2 describe-security-groups --group-ids sg-xxx --query 'SecurityGroups[0].IpPermissions'

# Check network policy
kubectl get networkpolicy -A
kubectl describe networkpolicy -A
```

### ALB/NLB Health Check Failure

```bash
# Check Ingress
kubectl get ingress
kubectl describe ingress <ingress-name>

# Check target groups and health
aws elbv2 describe-target-groups --query "TargetGroups[?contains(TargetGroupName,'k8s')]"
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# Verify readiness probe
kubectl get endpoints <service>
kubectl describe svc <service>
```

---

## IAM Issues

### AccessDenied

```bash
# Simulate IAM permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/<user> \
  --action-names eks:DescribeCluster

# Check CloudTrail for EKS deny
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DescribeCluster \
  --start-time $(date -d '-1 hour' +%Y-%m-%dT%H:%M:%S)
```

### Pod Cannot Access AWS API

```bash
# Verify pod SA
kubectl get pod <pod> -o jsonpath='{.spec.serviceAccountName}'
kubectl get sa <sa-name> -n <namespace> -o yaml

# Check IRSA role annotation
kubectl get sa <sa-name> -n <namespace> -o jsonpath='{.metadata.annotations}'

# Verify IAM role trust policy
aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'

# Test credential from pod
kubectl exec -it <pod> -- aws sts get-caller-identity
```

### IRSA Issues

```bash
# Check OIDC provider
aws iam list-open-id-connect-providers

# Verify issuer in trust policy matches cluster
aws eks describe-cluster --name my-cluster --query "cluster.identity.oidc.issuer"

# Check web identity token file
kubectl exec -it <pod> -- ls /var/run/secrets/eks.amazonaws.com/serviceaccount/
```

### Pod Identity Issues

```bash
# Check Pod Identity Agent
kubectl get pods -n kube-system | grep "eks-pod-identity-agent"

# Check association
aws eks list-pod-identity-associations --cluster-name my-cluster

# Verify Pod Identity association
aws eks describe-pod-identity-association \
  --cluster-name my-cluster \
  --association-id <id>
```

---

## Storage Issues

### PVC Pending

```bash
# Describe PVC
kubectl describe pvc <pvc-name> -n <namespace>

# Check StorageClass
kubectl get sc
kubectl describe sc <sc-name>

# Check storage capacity/quota
kubectl get resourcequota -n <namespace>
```

### Mount Failures

```bash
# Describe pod (check events for mount errors)
kubectl describe pod <pod-name> -n <namespace> | grep -i "mount\|volume"

# Check CSI driver pods
kubectl get pods -n kube-system | grep "ebs-csi\|efs-csi"

# Check volume permissions
kubectl exec -it <pod> -- ls -la /mnt
```

### EBS Volume Attachment Issues

```bash
# Describe PV for volume ID
kubectl describe pv | grep -A3 "VolumeAttributes"

# Check EBS volume status
aws ec2 describe-volumes --volume-ids <vol-id> \
  --query 'Volumes[0].{Status:State,Attachments:Attachments}'

# Verify node is in same AZ as volume
aws ec2 describe-instances --instance-ids <node-id> \
  --query 'Reservations[0].Instances[0].{AZ:Placement.AvailabilityZone}'
```

### EFS Mount Issues

```bash
# Check EFS mount target
aws efs describe-mount-targets --file-system-id fs-xxxxxxx

# Check VPC connectivity (EFS requires mount targets in same VPC)
aws efs describe-mount-target-security-groups --mount-target-id fsmt-xxxx

# NFS test from pod
kubectl exec -it <pod> -- mount | grep nfs
kubectl exec -it <pod> -- df -h
```

---

## Helm Troubleshooting

```bash
# List releases
helm list -A

# Get release status
helm status <release> -n <namespace>

# Get values
helm get values <release> -n <namespace>

# History (for rollback)
helm history <release> -n <namespace>

# Validate manifest (dry run)
helm template <release> ./chart -n <namespace>

# Rollback
helm rollback <release> <revision> -n <namespace>
```

---

## Command Reference by Category

| Category | Command |
|----------|---------|
| Cluster | `aws eks describe-cluster`, `kubectl cluster-info` |
| API server | `kubectl get --raw /healthz` |
| Auth | `aws sts get-caller-identity`, `kubectl auth can-i` |
| Nodes | `kubectl get/describe nodes`, `kubectl top node` |
| Pods | `kubectl describe pod`, `kubectl logs` |
| Networking | `kubectl get endpoints`, `kubectl exec` |
| IAM | `aws iam simulate-*`, `aws sts` |
| Storage | `kubectl get pvc/pv`, `kubectl describe sc` |
| Logs | `aws logs filter-log-events` |
| Helm | `helm status/history/list` |

---

## References

- [Troubleshoot EKS](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html)
- [Kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [EKS troubleshooting VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting-vpc-cni.html)