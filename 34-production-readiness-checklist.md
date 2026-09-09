# 34. Production Readiness Checklist

## Overview

A comprehensive production readiness checklist for EKS clusters and the applications running on them. Use this as a pre-launch / periodic review gate. Each item includes the best-practice requirement and where to verify it.

---

## Infrastructure & Cluster

- [ ] Cluster control plane running a **supported EKS version** (EKS Standard Support), with upgrade path documented. Verify: `aws eks describe-cluster --name <cluster>`
- [ ] Control plane logging (api/audit/authenticator/controllerManager/scheduler) enabled. Verify: `aws eks describe-cluster` `logging`
- [ ] EKS cluster **encryption enabled** (KMS envelope encryption on secrets). Verify: `aws eks describe-cluster --query cluster.encryptionConfig`
- [ ] Cluster endpoint configured with **private access PLUS restricted public CIDR** (or private-only). Verify: `resourcesVpcConfig.endpointPublicAccess`
- [ ] Cluster is region/AZ redundant; control plane spans 3 AZs automatically.
- [ ] kubectl access uses **EKS Access Entries** (not legacy aws-auth), principals are IAM roles, not long-lived users.
- [ ] `kubeconfig` uses short-lived credentials; no static admin tokens.
- [ ] EKS add-ons (vpc-cni, kube-proxy, coredns, CSI drivers) pinned and auto-upgrade disabled for control.

## Networking

- [ ] VPC designed with sufficient CIDR headroom (secondary CIDRs reserved).
- [ ] **Prefix delegation** enabled for VPC CNI (better pod density / reduced IP pressure).
- [ ] VPC CNI `ENABLE_POD_ENI` / `ENABLE_IPv4` configured for application requirements.
- [ ] Public subnets only used for load balancers / NAT; private subnets host nodes.
- [ ] Security groups least-privilege (no 0.0.0.0/0 inbound, except via LB/WAF).
- [ ] DNS: CoreDNS autoscaled; node-local DNS cache considered.
- [ ] **No unnecessary cross-VPC networking**; VPC peering/Transit Gateway/PrivateLink used securely.
- [ ] All outbound accessible via NAT/Transit Gateway, not IGW for nodes.
- [ ] NetworkPolicy: default-deny namespace + explicit allow rules applied.

## Node / Compute

- [ ] Nodes use **EKS-optimized AMI or Bottlerocket** (no custom untested AMIs).
- [ ] Managed node groups or EKS Auto Mode (preferred) over self-managed.
- [ ] Nodes spread across >= 3 AZs.
- [ ] **Karpenter** or Cluster Autoscaler configured with min/max bounds.
- [ ] **Spot instances** evaluated for stateless workloads; PDBs configured for bursts.
- [ ] Node selectors, taints/tolerations, topology spread constraints used appropriately.
- [ ] `kubelet` maxPods correct for instance type; resource reservations reviewed.
- [ ] Node labels/taints documented; instance types tested for steady-state resource usage.

## Storage

- [ ] StorageClasses defined: gp3 (AZ), EFS (multi-AZ RWX), S3 for objects.
- [ ] Default StorageClass set and documented.
- [ ] **EBS CSI driver encrypted** (KMS), `encrypted: true` default StorageClass.
- [ ] EBS volumes use `volumeBindingMode: WaitForFirstConsumer` (non-blocking/zoned).
- [ ] ReadWriteOnce storage confined to single AZ (pod affinity set for zonal storage).
- [ ] **Backup plan in place**: EBS snapshots / Velero to S3, AWS Backup for EFS/RDS.
- [ ] Retention + restore-tested backups documented; RTO/RPO defined.

## Security

- [ ] **EKS Access Entries** enforced; aws-auth deprecated/removed.
- [ ] RBAC least privilege; no permanent `cluster-admin` for humans.
- [ ] **Pod Identity** or IRSA used for pod-level AWS permissions; no instance-profile-wide permissions.
- [ ] Pod Security Standards: delete default privileged; use `restricted` for workloads.
- [ ] Seccomp/AppArmor contexts where applicable; no privileged containers.
- [ ] Images: signed / pinned by digest; private registry (ECR) enforced via admission/policy.
- [ ] Image scanning enabled (ECR + Inspector) with critical/high gating.
- [ ] **GuardDuty** (EKS Runtime/Protection) + **Security Hub** enabled and configured with findings → alerts.
- [ ] **CloudTrail** enabled organization-wide; retention aligned to audit policy.
- [ ] Secrets not in ConfigMaps; secrets via External Secrets + Secrets Manager/SSM with rotation.
- [ ] KMS keys *Customer Managed*, not AWS-managed, with access key rotation / fine-grained grants.
- [ ] mTLS/service mesh evaluated for sensitive workloads.
- [ ] Secrets scanned for leaks (git pre-commit / secret scanners).

## Application / Workloads

- [ ] All pods define **resource requests AND limits**; no unbounded pods.
- [ ] **HPA** configured for primary workloads (CPU/memory/custom/Prometheus).
- [ ] Workloads have **liveness AND readiness probes** (no default exec absent design).
- [ ] Replicas >= 2 (>= 3 recommended) with **PodDisruptionBudget**.
- [ ] `terminationGracePeriodSeconds` set; graceful shutdown handled (SIGTERM → drain).
- [ ] Pods set `topologySpreadConstraints` / `podAntiAffinity` across AZs where beneficial.
- [ ] Stateful workloads use StatefulSet/PVC; stateless use Deployments.
- [ ] Applications authenticated via ServiceAccount; no hardcoded credentials in images.
- [ ] Config externalized (ConfigMaps vs Secrets) — secrets never in ConfigMaps.
- [ ] **No PVCs on node-local storage** unless intentional with backup.
- [ ] Idle pods terminated (HPA minReplicas justified).

## Ingress / Load Balancing

- [ ] AWS Load Balancer Controller current version; IAM policy matches controller version.
- [ ] Ingress uses ALB (L7) or NLB (L4) per traffic needs; **WAF** attached for internet-facing ALB.
- [ ] TLS via ACM (auto-renewal) at ALB; minimum TLS 1.2.
- [ ] Ingress annotations documented (target-type, backend-protocol, healthcheck).
- [ ] NLB cross-zone + connection draining reviewed.
- [ ] No `service.beta.kubernetes.io/aws-load-balancer-*` deprecated annotations on new resources.
- [ ] External traffic through WAF → ALB → service → pod (request routing documented).

## Observability

- [ ] **Container Insights** enabled (CloudWatch). Verify: `/aws/eks/<cluster>` log groups exist.
- [ ] **Prometheus + Grafana** deployed (kube-prometheus-stack / AMP); custom dashboards for critical services.
- [ ] Logging: Fluent Bit → CloudWatch/OpenSearch; structured JSON logging in apps; **central log aggregation**, retention set.
- [ ] Distributed tracing (X-Ray / OpenTelemetry / ADOT) for key services.
- [ ] Critical **alerts** configured and tested: high CPU/memory, pod pending/crashloop, node NotReady, HPA maxed, PVC 80% full, backend 5xx.
- [ ] PagerDuty/Slack notification routing defined; on-call documented.
- [ ] Metrics alter thresholds tuned (avoid noise); reduce false positives/postmortem.

## CI/CD & GitOps

- [ ] CI pipeline: lint → build → scan → push immutable (`git SHA`) images to ECR.
- [ ] **Test-first gate**: critical/high vulnerabilities block promotion.
- [ ] GitOps (Argo CD / Flux) or reviewed pipeline used; **manual applies only for emergency**.
- [ ] Rollback strategy exercised (Helm rollback / Git revert + sync).
- [ ] Access separation: dev vs prod clusters/IAM; promotions only via pipeline (no manual kubectl).
- [ ] ECR lifecycle policies (retain N tags) and replication (cross-region if DR).
- [ ] Secret scanning on CI (e.g., Gitleaks) + Dependabot/Snyk for dependency CVEs.

## Reliability & HA

- [ ] Multi-AZ/service spread; single points of failure identified and mitigated.
- [ ] Data store HA: RDS/Aurora Multi-AZ, ElastiCache replicas (not single-stateful deployment in cluster).
- [ ] Node failure tolerated (PDB + ASG + Karpenter).
- [ ] **DR plan documented + tested**: Velero restore, EBS/RDS restore, DNS failover (Route 53), runbook.
- [ ] RTO/RPO defined and tested quarterly.
- [ ] Cluster upgrade tested in non-prod (upgrade pre-check, node replacement).
- [ ] Route 53 weighted/failover routing configured (if DR traffic shift).

## Operations & Cost

- [ ] **Upgrade runbook** (control plane → add-ons → nodes) documented and rehearsed.
- [ ] Cost tagging (tag: Team/Env/Cost-Center) on all resources; **Cost Explorer/CUR** budgets set. Verify: `TagSpecifications` on nodes/LB/ECR/etc.
- [ ] Nat Gateway per-AZ analysis; remove unused NAT GWs in single-AZ scenarios.
- [ ] EBS snapshot + ECR lifecycle policies review; unused AMIs/EIPs/snapshots cleaned.
- [ ] Scale testing / load-testing baseline documented before launch.
- [ ] Backup/restore, upgrade, failover documented runbooks rehearsed.
- [ ] 3rd-party / open-source components versions pinned and licenses understood.

---

## Final Sign-off

- [ ] Last production-deployment pass executed successfully with no blocking issues.
- [ ] On-call escalation + contacts documented.
- [ ] Go / No-go decision recorded.

---

## References

- [EKS Best Practices](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)
- [EKS Optimized Images & releases](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)
- [EKS Security best practices](https://docs.aws.amazon.com/eks/latest/userguide/security.html)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [EKS Release Calendar & Extended Support](https://docs.aws.amazon.com/eks/latest/userguide/extended-support.html)