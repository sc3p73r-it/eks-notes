# 4. EKS Networking Deep Dive

## Overview

Networking in EKS is built on Amazon VPC. The VPC CNI plugin assigns real VPC IP addresses to pods, making them first-class citizens in the VPC network. This section covers VPC design, VPC CNI internals, Kubernetes networking, AWS load balancing, and network policies.

---

## Amazon VPC Design

### VPC Architecture for EKS

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16 = 65,536 IPs)                       │
│                                                                         │
│  ┌───────────────────────────────┐  ┌───────────────────────────────┐  │
│  │  Public Subnet (10.0.0.0/20)  │  │  Private Subnet (10.0.32.0/20)│  │
│  │  AZ-1a                        │  │  AZ-1a                         │  │
│  │                               │  │                                │  │
│  │  ┌─────────────────────────┐  │  │  ┌──────────────────────────┐ │  │
│  │  │ Internet Gateway (IGW)  │  │  │  │ NAT Gateway              │ │  │
│  │  └─────────────────────────┘  │  │  └──────────────────────────┘ │  │
│  │  ┌─────────────────────────┐  │  │  ┌──────────────────────────┐ │  │
│  │  │ ALB / NLB               │  │  │  │ EKS Node Group           │ │  │
│  │  └─────────────────────────┘  │  │  │ ┌──────┐ ┌──────┐       │ │  │
│  └───────────────────────────────┘  │  │ │Node 1│ │Node 2│       │ │  │
│                                     │  │ └──────┘ └──────┘       │ │  │
│  ┌───────────────────────────────┐  │  └──────────────────────────┘ │  │
│  │  Public Subnet (10.0.16.0/20) │  │                                │  │
│  │  AZ-1b                        │  │  ┌───────────────────────────────┐  │
│  │                               │  │  │ Private Subnet (10.0.48.0/20) │  │
│  │  ┌─────────────────────────┐  │  │  │ AZ-1b                         │  │
│  │  │ ALB / NLB               │  │  │  │                                │  │
│  │  └─────────────────────────┘  │  │  │  ┌──────────────────────────┐ │  │
│  └───────────────────────────────┘  │  │  │ EKS Node Group           │ │  │
│                                     │  │  │ ┌──────┐ ┌──────┐       │ │  │
│  ┌───────────────────────────────┐  │  │  │ │Node 3│ │Node 4│       │ │  │
│  │  Public Subnet (10.0.32.0/20) │  │  │  │ └──────┘ └──────┘       │ │  │
│  │  AZ-1c                        │  │  │  └──────────────────────────┘ │  │
│  │                               │  │  └───────────────────────────────┘  │
│  │  ┌─────────────────────────┐  │  │                                    │
│  │  │ ALB / NLB               │  │  │  ┌───────────────────────────────┐  │
│  │  └─────────────────────────┘  │  │  │ Private Subnet (10.0.64.0/20) │  │
│  └───────────────────────────────┘  │  │ AZ-1c                         │  │
│                                     │  │                                │  │
│  ┌─────────────────────────────────┐│  │  ┌──────────────────────────┐ │  │
│  │  VPC Endpoints                  ││  │  │ EKS Node Group           │ │  │
│  │  ┌──────┐ ┌──────┐ ┌──────┐   ││  │  │ ┌──────┐ ┌──────┐       │ │  │
│  │  │  S3  │ │  ECR │ │  KMS │   ││  │  │ │Node 5│ │Node 6│       │ │  │
│  │  │(GW)  │ │(IF)  │ │(IF)  │   ││  │  │ └──────┘ └──────┘       │ │  │
│  │  └──────┘ └──────┘ └──────┘   ││  │  └──────────────────────────┘ │  │
│  └─────────────────────────────────┘│  └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### CIDR Planning

| Subnet | CIDR | IPs | AZ | Purpose |
|--------|------|-----|-----|---------|
| Public-A | 10.0.0.0/20 | 4,096 | AZ-1a | ALB, NAT GW |
| Public-B | 10.0.16.0/20 | 4,096 | AZ-1b | ALB |
| Public-C | 10.0.32.0/20 | 4,096 | AZ-1c | ALB |
| Private-A | 10.0.48.0/20 | 4,096 | AZ-1a | Nodes |
| Private-B | 10.0.64.0/20 | 4,096 | AZ-1b | Nodes |
| Private-C | 10.0.80.0/20 | 4,096 | AZ-1c | Nodes |
| Pod CIDR (VPC CNI) | 10.0.0.0/16 | Shared with VPC | All | Pod IPs |

**Important**: VPC CNI assigns pod IPs from the node subnet's secondary CIDR blocks. Plan subnet CIDRs large enough for both nodes and pods.

### VPC Endpoints

| Endpoint Type | Service | Purpose | Cost |
|---------------|---------|---------|------|
| Gateway | S3 | S3 access without NAT | Free |
| Gateway | DynamoDB | DynamoDB access without NAT | Free |
| Interface | ECR Docker | Image pulls | Per-hour + data |
| Interface | ECR API | Registry API calls | Per-hour + data |
| Interface | STS | IAM credentials | Per-hour + data |
| Interface | KMS | Encryption key operations | Per-hour + data |
| Interface | Secrets Manager | Secret retrieval | Per-hour + data |
| Interface | CloudWatch | Logs and metrics | Per-hour + data |
| Interface | API Gateway | API management | Per-hour + data |
| Interface | Elastic Load Balancing | LB management | Per-hour + data |

```bash
# Create VPC endpoint for ECR (Docker)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123456789abcdef0 \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-0abc123 subnet-0def456 \
  --security-group-ids sg-0123456789abcdef0

# Create VPC endpoint for S3 (Gateway)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0123456789abcdef0 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-0123456789abcdef0
```

---

## Amazon VPC CNI

### How Pods Receive IP Addresses

The VPC CNI plugin assigns real VPC IP addresses to pods using Elastic Network Interfaces (ENIs) and secondary IP addresses.

```
┌──────────────────────────────────────────────────────────────┐
│                         Node (EC2)                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Primary ENI                         │   │
│  │  Primary IP: 10.0.32.10                               │   │
│  │  Secondary IPs: 10.0.32.11, 10.0.32.12, ...          │   │
│  │                                                        │   │
│  │  ┌────────────────┐  ┌────────────────┐              │   │
│  │  │  Pod A          │  │  Pod B          │              │   │
│  │  │  IP: 10.0.32.11│  │  IP: 10.0.32.12│              │   │
│  │  │  (secondary IP)│  │  (secondary IP)│              │   │
│  │  └────────────────┘  └────────────────┘              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Secondary ENI                       │   │
│  │  Primary IP: 10.0.32.20                               │   │
│  │  Secondary IPs: 10.0.32.21, 10.0.32.22, ...          │   │
│  │                                                        │   │
│  │  ┌────────────────┐  ┌────────────────┐              │   │
│  │  │  Pod C          │  │  Pod D          │              │   │
│  │  │  IP: 10.0.32.21│  │  IP: 10.0.32.22│              │   │
│  │  │  (secondary IP)│  │  (secondary IP)│              │   │
│  │  └────────────────┘  └────────────────┘              │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### ENI and Secondary IP Allocation

| Instance Type | ENIs | IPs per ENI | Total IPs | Max Pods |
|---------------|------|-------------|-----------|----------|
| m5.large | 3 | 10 | 30 | 29 |
| m5.xlarge | 4 | 15 | 60 | 58 |
| m5.2xlarge | 4 | 15 | 60 | 58 |
| m5.4xlarge | 4 | 30 | 120 | 118 |
| c5.large | 3 | 10 | 30 | 29 |
| r5.large | 3 | 10 | 30 | 29 |

### Prefix Delegation

Prefix delegation assigns /28 prefixes (16 IPs) instead of individual secondary IPs, dramatically increasing the number of pods per node:

```bash
# Enable prefix delegation
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true

# Check allocated prefixes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDRs}{"\n"}{end}'
```

| Mode | IPs per ENI | Max Pods (m5.large) |
|------|-------------|---------------------|
| Secondary IPs | 10 | 29 |
| Prefix Delegation | 16 per prefix | 110+ |

### IP Exhaustion Solutions

1. **Prefix delegation**: More IPs per node
2. **Larger subnets**: /18 or /16 CIDR blocks
3. **Custom networking**: Use separate subnets for pods
4. **Secondary CIDR blocks**: Add additional CIDR to VPC
5. **NAT Gateway for pods**: Route pod traffic through NAT

### Security Groups for Pods

```yaml
# Pod with specific security group
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: my-sg-policy
spec:
  podSelector:
    matchLabels:
      app: my-app
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/16
```

---

## Kubernetes Networking

### Pod-to-Pod Communication

**Same node:**
```
Pod A (10.0.32.11) → kube-proxy (iptables) → Pod B (10.0.32.12)
```

**Cross-node:**
```
Pod A (10.0.32.11) → VPC routing → Node 2 (10.0.32.20) → Pod C (10.0.32.21)
```

### Service Networking

| Service Type | ClusterIP | NodePort | LoadBalancer | ExternalName |
|-------------|-----------|----------|-------------|--------------|
| Purpose | Internal only | Node access | External access | DNS alias |
| Scope | Cluster | Node | External | External |
| Cost | Free | Free | Per LB | Free |
| Example | Backend API | Debug | Web frontend | External DB |

```yaml
# ClusterIP Service
apiVersion: v1
kind: Service
metadata:
  name: my-backend
spec:
  type: ClusterIP
  selector:
    app: my-backend
  ports:
    - port: 80
      targetPort: 8080

# NodePort Service
apiVersion: v1
kind: Service
metadata:
  name: my-backend-nodeport
spec:
  type: NodePort
  selector:
    app: my-backend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080

# LoadBalancer Service
apiVersion: v1
kind: Service
metadata:
  name: my-frontend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: my-frontend
  ports:
    - port: 80
      targetPort: 8080
```

---

## AWS Load Balancing

### ALB vs NLB

| Feature | ALB | NLB |
|---------|-----|-----|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP) |
| Protocol | HTTP, HTTPS | TCP, UDP, TLS |
| Static IP | No | Yes |
| WebSocket | Yes | Yes |
| Path-based routing | Yes | No |
| Host-based routing | Yes | No |
| WAF integration | Yes | No |
| Health checks | HTTP, TCP | TCP, HTTP |
| Target types | IP, Instance | IP, Instance |
| Cross-zone | Yes | Yes (configurable) |
| Use case | Web apps, microservices | TCP apps, static IP, gaming |

### AWS Load Balancer Controller

```bash
# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1
```

### Ingress with ALB

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc-123
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:123456789012:regional/webacl/my-acl/abc-123
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Gateway API

Gateway API is the successor to Ingress, providing more expressive and role-oriented configuration:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: aws
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-tls-cert
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "app.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 80
```

---

## Network Policies

```yaml
# Deny all ingress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Allow frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
---
# Allow backend to external (HTTPS only)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-external
spec:
  podSelector:
    matchLabels:
      app: backend
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - port: 443
          protocol: TCP
```

**Note**: EKS does not natively enforce NetworkPolicy. You need Calico or Cilium installed as a CNI or policy engine.

---

## DNS Configuration

```yaml
# Custom CoreDNS ConfigMap
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
    # Custom DNS for internal services
    example.internal:53 {
        forward . 10.0.0.2
    }
```

---

## Service Mesh Comparison

| Feature | App Mesh | Istio | Linkerd | Cilium |
|---------|----------|-------|---------|--------|
| Data plane | Envoy | Envoy | linkerd2-proxy | eBPF |
| Configuration | CRDs | CRDs | CRDs | CRDs |
| mTLS | Yes | Yes | Yes | Yes |
| Traffic management | Basic | Advanced | Basic | Advanced |
| Observability | Basic | Advanced | Advanced | Advanced |
| Resource usage | Medium | High | Low | Low |
| Complexity | Low | High | Medium | Medium |
| AWS integration | Native | Third-party | Third-party | Third-party |
| Status | Deprecated (2024) | Active | Active | Active |

---

## Troubleshooting

```bash
# Check VPC CNI pods
kubectl get pods -n kube-system -l k8s-app=aws-node

# Check CNI configuration
kubectl get daemonset aws-node -n kube-system -o yaml

# Check node IP allocation
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'

# Check pod IPs
kubectl get pods -o wide -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP

# Check ENIs on a node
aws ec2 describe-network-interfaces --filters Name=instance-id,Values=i-0123456789abcdef0

# Test DNS resolution
kubectl run dns-test --image=busybox --rm -it -- nslookup kubernetes.default

# Test pod connectivity
kubectl exec -it <pod1> -- ping <pod2-ip>

# Check VPC CNI logs
kubectl logs -n kube-system <aws-node-pod> -c aws-node

# Check service endpoints
kubectl get endpoints <service-name>

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

## References

- [VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)
- [Network Policies](https://docs.aws.amazon.com/eks/latest/best-practices/network-policies.html)
- [AWS Load Balancer Controller](https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html)
- [Security Groups for Pods](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
