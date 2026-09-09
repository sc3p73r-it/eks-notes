# 9. EKS Ingress and Application Traffic

## Traffic Flow Overview

This section traces the complete path of a request from the internet to a Kubernetes pod and back, covering every component in the chain.

---

## Primary Traffic Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Internet → Pod Traffic Flow                           │
│                                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │
│  │ Internet │───▶│ Route 53 │───▶│CloudFront│───▶│  AWS WAF │        │
│  │          │    │  (DNS)   │    │  (CDN)   │    │          │        │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘        │
│                                                         │              │
│                                                         ▼              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐        │
│  │   Pod    │◀───│Kubernetes│◀───│Kubernetes│◀───│   ALB    │        │
│  │          │    │ Service  │    │ Ingress  │    │ (Public) │        │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### Route 53 (DNS)

```bash
# Create hosted zone
aws route53 create-hosted-zone --name example.com --caller-reference $(date +%s)

# Create alias record pointing to ALB
aws route53 change-resource-record-sets --hosted-zone-id Z1234567890 --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "AliasTarget": {
        "DNSName": "k8s-default-myingress-abc123.us-east-1.elb.amazonaws.com",
        "HostedZoneId": "Z35SXDOTRQ7X7K",
        "EvaluateTargetHealth": true
      }
    }
  }]
}'
```

### CloudFront (CDN)

```bash
# Create CloudFront distribution
aws cloudfront create-distribution --distribution-config '{
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "eks-alb",
      "DomainName": "k8s-default-myingress-abc123.us-east-1.elb.amazonaws.com",
      "CustomOriginConfig": {
        "HTTPPort": 80,
        "HTTPSPort": 443,
        "OriginProtocolPolicy": "https-only"
      }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "eks-alb",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/abc-123",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "Enabled": true,
  "Aliases": {
    "Quantity": 1,
    "Items": ["app.example.com"]
  }
}'
```

### AWS WAF

```bash
# Create WAFv2 Web ACL
aws wafv2 create-web-acl \
  --name eks-web-acl \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "RateLimit",
      "Priority": 1,
      "Action": {"Block": {}},
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimit"
      }
    },
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 2,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSet"
      }
    }
  ]' \
  --visibility-config '{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"eks-web-acl"}'
```

### ALB (Application Load Balancer)

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
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
spec:
  tls:
    - hosts:
        - app.example.com
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

### Kubernetes Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

### Kubernetes Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:v1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 1
              memory: 512Mi
```

---

## Alternative Architectures

### Internal ALB (Private Services)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: internal-service
                port:
                  number: 80
```

### NLB (TCP/Static IP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nlb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

---

## TLS and ACM

### Certificate Request

```bash
# Request certificate
aws acm request-certificate \
  --domain-name app.example.com \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS

# Get certificate ARN
aws acm list-certificates --query "CertificateSummaryList[?DomainName=='app.example.com'].CertificateArn" --output text
```

---

## Health Checks

| Component | Health Check | Purpose |
|-----------|-------------|---------|
| ALB | HTTP/TCP | Route traffic to healthy targets |
| Kubernetes | Readiness/LB | Determine pod readiness |
| Kubernetes | Liveness/LB | Restart unhealthy containers |
| Kubernetes | Startup/LB | Allow slow-starting containers |

---

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| TLS | Use TLS everywhere, redirect HTTP to HTTPS |
| WAF | Enable AWS WAF with managed rule groups |
| Health checks | Configure appropriate thresholds |
| Security groups | Restrict ALB security groups |
| Internal services | Use internal ALB for backend services |
| Rate limiting | Implement rate limiting via WAF |
| Monitoring | Monitor ALB metrics in CloudWatch |

---

## Troubleshooting

```bash
# Check Ingress
kubectl get ingress
kubectl describe ingress <ingress-name>

# Check ALB target group
aws elbv2 describe-target-groups --query "TargetGroups[?contains(TargetGroupName,'k8s')].TargetGroupArn"

# Check target health
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/k8s-default-myservice/abc123

# Check security groups
aws ec2 describe-security-groups --filters Name=group-name,Values=k8s-elb*

# Check ALB access logs
aws s3 ls s3://my-alb-logs/AWSLogs/123456789012/elasticloadbalancing/
```

---

## References

- [AWS Load Balancer Controller](https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter-managing.html)
