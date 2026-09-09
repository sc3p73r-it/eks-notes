# 35. References

## Overview

Curated list of official AWS, Kubernetes, and partner references for EKS operations. Grouped by topic.

---

## AWS EKS Core Docs

- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [EKS Architecture Options](https://docs.aws.amazon.com/eks/latest/userguide/eks-compute.html)
- [EKS Release Calendar (versions)](https://docs.aws.amazon.com/eks/latest/userguide/platform-versions.html)
- [EKS Extended Support](https://docs.aws.amazon.com/eks/latest/userguide/extended-support.html)
- [EKS Access Entries vs aws-auth](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [IRSA (IAM Roles for Service Accounts)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [EKS Capabilities (managed ACK/Argo CD/kro)](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-eks-capabilities/)
- [EKS Hybrid Nodes](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes.html)
- [EKS Optimized AMIs](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)
- [EKS Node Groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)
- [EKS Control Plane Logging](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html)
- [EKS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)
- [EKS Anywhere](https://anywhere.eks.amazonaws.com/)
- [EKS Windows Containers](https://docs.aws.amazon.com/eks/latest/userguide/windows-support.html)

## Networking

- [Amazon VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/networking.html)
- [VPC CNI prefix delegation](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Gateway API on EKS](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [NetworkPolicy docs](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

## Security

- [EKS Security](https://docs.aws.amazon.com/eks/latest/userguide/security.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [EKS Encryption Provider](https://docs.aws.amazon.com/eks/latest/userguide/encryption.html)
- [Amazon GuardDuty for EKS](https://docs.aws.amazon.com/guardduty/latest/ug/monitoring-eks.html)
- [Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/)
- [External Secrets Operator](https://external-secrets.io/latest/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [EKS Best Practices (GitHub)](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)

## Storage

- [EBS CSI driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [EFS CSI driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)
- [Amazon EFS documentation](https://docs.aws.amazon.com/efs/latest/userguide/whatisefs.html)
- [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)
- [Velero](https://velero.io/docs/)

## Scaling

- [Karpenter](https://karpenter.sh/docs/)
- [Cluster Autoscaler on AWS](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [KEDA (event-driven autoscaling)](https://keda.sh/)
- [Cluster Proportional Autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler)
- [Amazon EC2 Spot](https://aws.amazon.com/ec2/spot/)

## Observability

- [Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)
- [Amazon Managed Prometheus](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html)
- [Amazon Managed Grafana](https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html)
- [OpenTelemetry docs](https://opentelemetry.io/docs/)
- [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
- [Fluent Bit](https://docs.fluentbit.io/manual/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [AWS Observability Accelerator (Terraform)](https://aws-observability.github.io/terraform-aws-observability-accelerator/)
- [CDK AWS Observability Accelerator](https://aws-observability.github.io/cdk-aws-observability-accelerator/)
- [Kubecost (K8s cost visibility)](https://www.kubecost.com/)
- [One Observability Workshop](https://catalog.workshops.aws/observability/en-US)

## CI/CD

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/)
- [Flux](https://fluxcd.io/)
- [ACK (AWS Controllers for Kubernetes)](https://aws-controllers-k8s.github.io/community/)
- [Crossplane](https://www.crossplane.io/)
- [kro (Kube Resource Orchestrator)](https://kro.run/)
- [Kyverno (policy/security admission)](https://kyverno.io/)
- [Helm](https://helm.sh/docs/)
- [Kustomize](https://kustomize.io/)
- [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)
- [Amazon ECR scanning](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)

## Terraform

- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [terraform-aws-eks module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
- [Terraform EKS Blueprints](https://aws-ia.github.io/terraform-aws-eks-blueprints/)
- [Terraform docs](https://developer.hashicorp.com/terraform/docs)

## Cost

- [AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [EKS Pricing](https://aws.amazon.com/eks/pricing/)
- [Savings Plans](https://aws.amazon.com/savingsplans/)
- [AWS Pricing Calculator](https://calculator.aws/)

## Reference Architectures

- [EKS Best Practices (AWS cloudformation-templates guide)](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [AWS Well-Architected — Kubernetes workloads](https://docs.aws.amazon.com/wellarchitected/latest/kubernetes-workloads/welcome.html)
- [EKS Blueprints for Terraform](https://github.com/aws-ia/terraform-aws-eks-blueprints)
- [Amazon EKS Workshop](https://www.eksworkshop.com/)
- [EKS Workshop GitHub (aws-samples/eks-workshop-v2)](https://github.com/aws-samples/eks-workshop-v2)
- [EKS Workshop — Essentials / Fast Paths](https://www.eksworkshop.com/docs/fastpaths/)
- [Kiro (natural language EKS operations)](https://github.com/awslabs/kiro)
- [AWS Containers Blog](https://aws.amazon.com/blogs/containers/)

---

## Maintenance

This references list should be reviewed monthly for:
- New EKS versions (Standard/Extended support changes)
- Changes to `aws-auth`/Access Entry guidance
- VPC CNI / LB controller features and deprecation annotations
- Karpenter/Cluster Autoscaler release notes