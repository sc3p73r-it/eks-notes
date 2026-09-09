# 32. Command Reference

## Overview

Practical command cheat sheet for AWS CLI, kubectl, helm, eksctl, terraform, and docker commands relevant to EKS operation.

---

## Cluster

```bash
# AWS CLI
aws eks create-cluster --name <cluster> --role-arn <arn> --resources-vpc-config subnetIds=<subnets>
aws eks describe-cluster --name <cluster>
aws eks list-clusters --region <region>
aws eks delete-cluster --name <cluster>
aws eks update-cluster-version --name <cluster> --kubernetes-version 1.32
aws eks update-kubeconfig --name <cluster> --region <region>

# eksctl
eksctl create cluster --name <cluster> --version 1.31 --region us-east-1 --nodegroup-name workers --node-type m5.large --nodes 3
eksctl get cluster --region us-east-1
eksctl delete cluster --name <cluster> --disable-nodegroup-eviction

# kubectl
kubectl cluster-info
kubectl config view --minify
kubectl get componentstatuses
```

---

## Nodes

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top node
kubectl get nodes --show-labels
kubectl label node <node-name> <label>=<value>
kubectl taint nodes <node-name> <key>=<value>:NoSchedule
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl cordon <node-name>
kubectl uncordon <node-name>

# AWS
aws eks list-nodegroups --cluster-name <cluster>
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng>
aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <ng>
aws eks create-nodegroup --cluster-name <cluster> --nodegroup-name <ng> --node-role <arn> --instance-types m5.large --scaling-config minSize=1,maxSize=10,desiredSize=3
```

---

## Pods

```bash
kubectl get pods -A
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --tail=100
kubectl logs <pod> -n <namespace> --previous
kubectl exec -it <pod> -n <namespace> -- /bin/sh
kubectl delete pod <pod> -n <namespace>
kubectl get pod <pod> -n <namespace> -o yaml
kubectl get pods --sort-by=.status.phase
kubectl top pods -n <namespace>
```

---

## Networking

```bash
# Services and Endpoints
kubectl get svc -A
kubectl describe svc <svc> -n <namespace>
kubectl get endpoints <svc> -n <namespace>

# Ingress/NetworkPolicy
kubectl get ingress -A
kubectl describe ingress <ingress> -n <namespace>
kubectl get networkpolicy -A

# DNS
kubectl run dnstest --image=busybox --rm -it -- nslookup kubernetes.default

# AWS
aws ec2 describe-security-groups --filters Name=tag:kubernetes.io/cluster/<cluster>,Values=owned
aws elbv2 describe-target-groups --query "TargetGroups[?contains(TargetGroupName,'k8s')]"
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws ec2 describe-network-interfaces --filters Name=instance-id,Values=<i-xxx>
```

---

## IAM

```bash
# Auth
aws sts get-caller-identity
kubectl auth can-i get pods -n <namespace> --as=<user>
kubectl auth can-i --list -n <namespace>

# Access entries
aws eks list-access-entries --cluster-name <cluster>
aws eks create-access-entry --cluster-name <cluster> --principal-arn <arn> --type STANDARD
aws eks associate-access-policy --cluster-name <cluster> --principal-arn <arn> --policy-arn <access-policy> --access-scope type=cluster
aws eks create-pod-identity-association --cluster-name <cluster> --namespace <ns> --service-account <sa> --role-arn <role-arn>
aws eks list-pod-identity-associations --cluster-name <cluster>

# IRSA
eksctl create iamserviceaccount --name <sa> --namespace <ns> --cluster <cluster> --attach-policy-arn <arn> --approve
eksctl utils associate-iam-oidc-provider --cluster <cluster> --approve
```

---

## Storage

```bash
kubectl get sc
kubectl get pv
kubectl get pvc -A
kubectl describe pvc <pvc> -n <namespace>
kubectl get storageclass <sc> -o yaml
kubectl patch pvc <pvc> -n <namespace> -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# AWS
aws ec2 describe-volumes --filters Name=tag:eks:cluster-name,Values=<cluster>
aws ec2 describe-snapshots --filters Name=tag:eks:cluster-name,Values=<cluster>
aws efs describe-file-systems
aws efs describe-mount-targets --file-system-id <fs-id>
```

---

## Logs

```bash
# Control plane logs
aws eks update-cluster-config --name <cluster> --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
aws logs describe-log-groups --log-group-name-prefix /aws/eks/<cluster>
aws logs filter-log-events --log-group-name /aws/eks/<cluster>/cluster --limit 10

# Container logs
kubectl logs <pod> -n <namespace> --tail=100

# Events
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Deployment

```bash
# Manifests
kubectl apply -f <file>
kubectl apply -k <kustomize-dir>
kubectl delete -f <file>
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout restart deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace>
kubectl set image deployment/<name> <container>=<image>:<tag> -n <namespace>

# Helm
helm repo add <name> <url>
helm search repo <name>
helm install <release> <chart> -n <namespace>
helm upgrade <release> <chart> -n <namespace> --set image.tag=v2
helm rollback <release> <revision> -n <namespace>
helm list -A
helm history <release> -n <namespace>
helm diff upgrade <release> <chart> -n <namespace> --detailed-previous
```

---

## Upgrades

```bash
# Cluster
aws eks update-cluster-version --name <cluster> --kubernetes-version 1.32
aws eks wait cluster-active --name <cluster>
aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <ng>

# Addons
aws eks list-addons --cluster-name <cluster>
aws eks create-addon --cluster-name <cluster> --addon-name <addon>
aws eks update-addon --cluster-name <cluster> --addon-name <addon> --resolve-conflicts OVERWRITE
aws eks describe-addon-versions --kubernetes-version 1.31

# kubectl drain nodes before replacement
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

---

## Docker

```bash
docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com < /dev/null
# (preferred: aws ecr get-login-password | docker login --username AWS --password-stdin <registry>)
docker build -t <registry>/<repo>:<tag> .
docker push <registry>/<repo>:<tag>
docker pull <registry>/<repo>:<tag>
docker images
docker rmi <image>
docker exec -it <container> <command>
docker ps
docker system prune
```

---

## ECR

```bash
aws ecr create-repository --repository-name <repo> --image-scanning-configuration scanOnPush=true
aws ecr describe-repositories
aws ecr describe-images --repository-name <repo>
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
aws ecr put-image-scanning-configuration --repository-name <repo> --image-scanning-configuration scanOnPush=true
aws ecr start-image-scan --repository-name <repo> --image-id imageTag=<tag>
aws ecr describe-image-scan-findings --repository-name <repo> --image-id imageTag=<tag>
aws ecr put-lifecycle-policy --repository-name <repo> --policy-text file://policy.json
```

---

## Terraform

```bash
terraform init
terraform validate
terraform fmt -recursive
terraform plan -out=tfplan
terraform apply tfplan
terraform show
terraform state list
terraform state rm <resource>
terraform import <address> <id>
terraform destroy
```

---

## Troubleshooting Quick References

| Issue | Command |
|-------|---------|
| Cluster down | `aws eks describe-cluster`, `kubectl cluster-info` |
| Node not ready | `kubectl describe node`, `journalctl -u kubelet` |
| Pod pending | `kubectl describe pod` (events) |
| CrashLoop | `kubectl logs --previous` |
| ImagePull | `kubectl describe pod` + `aws ecr get-login-password` |
| DNS | `kubectl run dnstest --image=busybox --rm -it -- nslookup kubernetes.default` |
| ing | `kubectl describe svc`, `aws elbv2 describe-target-health` |
| Auth | `aws sts get-caller-identity`, `kubectl auth can-i` |
| PVC | `kubectl describe pvc`, `aws ec2 describe-volumes` |

---

## References

- [AWS CLI EKS Reference](https://docs.aws.amazon.com/cli/latest/reference/eks/index.html)
- [Kubectl Cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [eksctl](https://eksctl.io/)
- [Helm Commands](https://helm.sh/docs/helm/)