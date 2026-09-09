# 16. EKS Security Scanning and DevSecOps

## Overview

A production DevSecOps pipeline integrates security scanning at every stage: source code (SAST), dependencies (SCA), secrets (secret scanning), container images, infrastructure as code (IaC), and runtime. Tools range from AWS-native (Inspector, GuardDuty, Security Hub) to third-party (Trivy, SonarQube, Snyk).

---

## DevSecOps Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DevSecOps Pipeline                                │
│                                                                         │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│  │Source│─▶│ SAST │─▶│ SCA  │─▶│Secret│─▶│Container│▶│  IaC │─▶│ Build│
│  │      │  │      │  │      │  │ Scan │  │  Scan │  │ Scan │  │      │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘
│     │          │         │         │         │         │         │
│     ▼          ▼         ▼         ▼         ▼         ▼         ▼
│  GitHub    Sonar     Dependabot  GitGuardian Trivy    Checkov    Docker
│  Actions   Qube      /Renovate  /trufflehog /ECR     /tfsec     Build
│   (300+)    (300+)    (SCA)
│                                                                         │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐        │
│  │ Deploy│─▶│ ECR  │─▶│ArgoCD│─▶│  EKS │─▶│Guard│─▶│ Sec  │        │
│  │      │  │      │  │      │  │      │  │Duty  │  │ Hub  │        │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘        │
│                                                                         │
│  Runtime Security: Amazon GuardDuty + Falco + CloudWatch                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Tool Classification

| Tool | Category | Type | AWS-Native? |
|------|----------|------|-------------|
| Amazon Inspector | Vulnerability scanning | AWS-native | Yes |
| ECR Basic Scanning | Container scanning | AWS-native | Yes |
| Amazon GuardDuty | Threat detection | AWS-native | Yes |
| AWS Security Hub | Security posture | AWS-native | Yes |
| Amazon CodeGuru | SAST | AWS-native | Yes |
| Trivy | Container scanning | Third-party (open source) | No |
| SonarQube | SAST | Third-party | No |
| GitHub Dependabot | SCA | Third-party (GitHub) | No |
| GitHub CodeQL | SAST | Third-party (GitHub) | No |
| GitGuardian | Secret scanning | Third-party | No |
| Snyk | SCA/SAST | Third-party | No |
| Checkov | IaC scanning | Third-party (open source) | No |
| tfsec | IaC scanning | Third-party (open source) | No |
| Falco | Runtime security | Third-party (open source) | No |

---

## Amazon Inspector (Enhanced ECR Scanning)

### Enable Inspector

```bash
# Enable Inspector
aws inspector2 enable --resource-types EC2 ECR EKS

# Check Inspector status
aws inspector2 list-findings --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"EKS"}]}'
```

### Auto-remediate findings

Attach a policy to the cross-account role to remediate via EventBridge:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: inspector-remediation
  namespace: security
data:
  remediation-target: |
    # Example: block a vulnerable image tag
    - finding: CRITICAL vulnerability in image
      action: deny image pull
```

---

## Amazon GuardDuty

### Enable EKS Protection

```bash
# Enable GuardDuty EKS protection
aws guardduty create-detector --enable

# Enable EKS audit log monitoring
aws guardduty update-detector \
  --detector-id <detector-id> \
  --features '[
    {"Name":"EKS_AUDIT_LOGS","Enabled":true},
    {"Name":"EKS_RUNTIME_MONITORING","Enabled":true,"Configuration":{"AuditEKSControlPlane":true}}
  ]'
```

### GuardDuty Findings

| Finding Type | Description |
|--------------|-------------|
| `Policy:Kubernetes/AnonymousAccessGranted` | High-privilege RBAC granted |
| `Policy:Kubernetes/AdminAccessToDefaultServiceAccount` | Admin SA used on default config |
| `CryptoCurrency:Kubernetes/BitcoinTool` | Crypto-mining credentials in cluster |
| `Execution:Kubernetes/ExecInPod` | `kubectl exec` unusual activity |
| `Backdoor:Kubernetes/CreateServiceAccount` | Suspicious SA creation |

---

## Third-Party Scanners in CI

### Trivy

```yaml
# GitHub Actions with Trivy
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
```

```bash
# Local scan
trivy image --severity CRITICAL,HIGH my-app:v1.2.3
trivy fs --severity CRITICAL,HIGH .
trivy k8s --cluster-name my-cluster
```

### Snyk

```bash
# Snyk container scan
snyk test --docker 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3

# Snyk code scan
snyk code test

# Snyk IaC scan
snyk iac test terraform/
```

### SonarQube

```yaml
# GitHub Actions with SonarQube
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@master
  with:
    args: >-
      -Dsonar.projectKey=my-app
      -Dsonar.sources=src
- name: SonarQube Quality Gate
  uses: SonarSource/sonarqube-quality-gate-action@master
```

### GitHub Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
```

### GitHub CodeQL

```yaml
# .github/workflows/codeql.yml
name: "CodeQL"
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript
      - name: Autobuild
        uses: github/codeql-action/autobuild@v3
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

### GitGuardian (Secret Scanning)

```yaml
# GitHub Actions with GitGuardian
- name: GitGuardian Private Scan
  uses: GitGuardian/ggshield-action@main
  with:
    args: 'sca scan ci'
```

---

## Security Scanning Configuration

### ECR Image Scanning (Basic)

```bash
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true
```

### Build-time Scanning in CI

```yaml
# GitHub Actions - Gate on ECR findings
- name: Get ECR scan findings
  run: |
    aws ecr start-image-scan --repository-name $REPO --image-id imageTag=$TAG --region $REGION
    until [ "$(aws ecr describe-image-scan-findings --repository-name $REPO --image-id imageTag=$TAG --query 'imageScanStatus.status' --output text)" = "COMPLETE" ]; do sleep 5; done
  env:
    REPO: ${{ env.ECR_REPOSITORY }}
    TAG: ${{ env.IMAGE_TAG }}
    REGION: ${{ env.AWS_REGION }}
```

---

## Runtime Security

### GuardDuty for Runtime

```bash
aws eks create-addon --cluster-name my-cluster --addon-name amazon-guardduty-agent
```

### Falco

```yaml
# Install Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set falco.syscallEventDrop="false"
```

Falco alerts on suspicious container runtime behavior:

```yaml
# Custom Falco rule
- rule: Interactive Shell in Container
  desc: An interactive shell was spawned in a container
  condition: >
    spawned_process and container
    and shell_procs and not proc.pname in (shell_binaries)
  output: >
    Shell spawned in container (user=%user.name container=%container.id shell=%proc.name parent=%proc.pname)
  priority: WARNING
```

---

## Security Hub Aggregation

```bash
# Enable Security Hub
aws securityhub enable-security-hub \
  --enable-standards '[
    {"StandardsArn":"arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.2.0"},
    {"StandardsArn":"arn:aws:securityhub:::standards/aws-foundational-security-best-practices/v/1.0.0"}
  ]'
```

---

## Production DevSecOps Best Practices

| Stage | Tool | Gate |
|-------|------|------|
| Source | GitHub CodeQL, SonarQube | Critical SAST findings block merge |
| Dependencies | Dependabot, Snyk | Critical SCA findings block merge |
| Secrets | GitGuardian, trufflehog | Any secret blocks merge |
| Container | Trivy, ECR scan | High+ vulnerabilities block deploy |
| IaC | Checkov, tfsec | High+ findings block apply |
| Runtime | GuardDuty, Falco | Real-time alerting |
| Compliance | Security Hub | Aggregate findings |

---

## References

- [Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html)
- [GuardDuty EKS Protection](https://docs.aws.amazon.com/guardduty/latest/ug/kubernetes-protection.html)
- [AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html)
- [Trivy](https://trivy.dev/)
- [Falco](https://falco.org/)
- [AWS EKS Best Practices - Security](https://aws.github.io/aws-cloudformation-templates/latest/other-guides/eks-best-practices.html)