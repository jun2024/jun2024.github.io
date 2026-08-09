---
title: "[Kubernetes] 로컬 K8s vs EKS (3) — 네트워킹: Service 노출, Ingress, DNS 비교"
excerpt: "Service를 외부에 어떻게 노출할까? NodePort/MetalLB vs AWS LB Controller, Nginx Ingress vs ALB, CoreDNS + Route53 연동까지 비교 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, EKS, AWS, Networking, Ingress, ALB, Route53, DevOps]

permalink: /kubernetes/local-vs-eks-3/

toc: true
toc_sticky: true

date: 2026-07-28
last_modified_at: 2026-07-28
---

> 이전 포스트 [로컬 vs EKS (2)](/kubernetes/local-vs-eks-2/)에서 설치 방법과 CNI 차이를 다뤘어요!

---

## 🦥 Service 노출 방식 비교

K8s에서 Service를 외부에 노출하는 방법이 환경에 따라 크게 달라요.

| Service Type | 로컬 K8s | EKS |
|-------------|---------|-----|
| **ClusterIP** | 동일 | 동일 |
| **NodePort** | 30000-32767 포트로 접근 | 동일하지만 SG 설정 필요 |
| **LoadBalancer** | 기본 미지원 (Pending) | **AWS NLB/CLB 자동 생성** |
| **ExternalName** | 동일 | 동일 |

### 로컬에서 LoadBalancer — MetalLB

로컬 K8s에서 `type: LoadBalancer` Service를 만들면 External IP가 **Pending** 상태로 멈춰요. 클라우드 LB가 없기 때문이에요.

```bash
# 로컬에서 LoadBalancer Service
kubectl get svc
# NAME    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)
# my-svc  LoadBalancer   10.96.0.100   <pending>     80:31234/TCP
```

**MetalLB**를 설치하면 로컬에서도 LoadBalancer를 사용할 수 있어요.

```bash
# MetalLB 설치
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

```yaml
# IP 풀 설정
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.250     # 사용할 IP 대역
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

### EKS에서 LoadBalancer — AWS LB 자동 생성

EKS에서는 `type: LoadBalancer`를 만들면 **AWS Load Balancer가 자동 생성**돼요.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
  annotations:
    # NLB 사용 (기본은 CLB)
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    # Internal LB (VPC 내부 전용)
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

| Annotation | 효과 |
|-----------|------|
| `aws-load-balancer-type: nlb` | Network Load Balancer 생성 |
| `aws-load-balancer-internal: true` | Internal LB (Private) |
| `aws-load-balancer-scheme: internet-facing` | Public LB |
| `aws-load-balancer-ssl-cert` | ACM SSL 인증서 연결 |

> 💡 **AWS Load Balancer Controller**를 설치하면 더 세밀한 제어가 가능해요. NLB IP 모드, Target Group 바인딩 등을 지원해요.

---

## 🦥 Ingress 비교

### 로컬 K8s — Nginx Ingress Controller

```bash
# Nginx Ingress Controller 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/baremetal/deploy.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
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
```

### EKS — AWS ALB Ingress Controller

EKS에서는 **AWS Load Balancer Controller**를 사용해서 Ingress를 **AWS ALB**로 구현해요.

```bash
# AWS Load Balancer Controller 설치 (Helm)
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  ingressClassName: alb
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
```

### Ingress Controller 비교

| 항목 | Nginx Ingress (로컬) | ALB Ingress (EKS) |
|------|--------------------|--------------------|
| L7 Load Balancer | Nginx Pod | AWS ALB |
| SSL/TLS | cert-manager + Let's Encrypt | **ACM (무료, 자동 갱신)** |
| WAF 연동 | 별도 구성 | AWS WAF 연동 가능 |
| 인증 | 별도 구성 | Cognito 연동 가능 |
| Target | Service (ClusterIP) | Pod IP 직접 (ip 모드) |
| 비용 | Pod 리소스만 | ALB 시간당 + LCU 과금 |

> 💡 **EKS 실무 팁**: `target-type: ip`를 사용하면 ALB가 Pod에 직접 트래픽을 보내요 (VPC CNI 덕분). NodePort를 거치지 않아서 홉이 줄고 성능이 좋아요.

---

## 🦥 DNS 비교

### 클러스터 내부 DNS — 동일

| 항목 | 로컬 K8s | EKS |
|------|---------|-----|
| DNS Server | CoreDNS | CoreDNS (Add-on) |
| Service DNS | `<svc>.<ns>.svc.cluster.local` | 동일 |
| Pod DNS | `<ip>.<ns>.pod.cluster.local` | 동일 |
| 설정 | ConfigMap (coredns) | 동일 |

클러스터 내부 DNS는 동일해요. CKA에서 배운 CoreDNS 지식이 그대로 적용돼요.

### 외부 DNS — Route53 연동

로컬에서는 외부 DNS를 수동으로 관리하지만, EKS에서는 **ExternalDNS**를 사용해서 Route53과 자동 연동할 수 있어요.

```bash
# ExternalDNS 설치 (Helm)
helm install external-dns bitnami/external-dns \
  --set provider=aws \
  --set domainFilters[0]=example.com \
  --set policy=sync \
  --set aws.zoneType=public
```

```yaml
# Ingress나 Service에 호스트를 지정하면
# ExternalDNS가 자동으로 Route53에 레코드 생성
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.example.com
spec:
  # ...
```

| 항목 | 로컬 K8s | EKS + ExternalDNS |
|------|---------|-------------------|
| 외부 DNS 등록 | 수동 (DNS 서버 직접 설정) | **자동** (Route53 레코드 생성) |
| 레코드 삭제 | 수동 | Ingress/Service 삭제 시 자동 삭제 |
| 와일드카드 | 수동 설정 | 지원 |
| TTL | 수동 설정 | annotation으로 설정 |

---

## 🦥 Security Group과 NetworkPolicy

### 로컬 K8s — NetworkPolicy만

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - port: 8080
```

### EKS — NetworkPolicy + Security Group for Pods

EKS에서는 K8s NetworkPolicy에 더해 **AWS Security Group을 Pod 레벨**로 적용할 수 있어요.

```yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: my-sg-policy
spec:
  podSelector:
    matchLabels:
      app: db
  securityGroups:
    groupIds:
      - sg-0123456789abcdef0     # RDS 접근용 SG
```

| 방법 | 범위 | 특징 |
|------|------|------|
| NetworkPolicy | Pod ↔ Pod | L3/L4, CNI가 지원해야 함 |
| Security Group for Pods | Pod ↔ AWS 리소스 | VPC 레벨 보안, RDS/ElastiCache 접근 제어 |

> 💡 **실무 핵심**: Pod에서 RDS에 접근할 때 Security Group for Pods를 사용하면 Pod 레벨로 DB 접근을 제어할 수 있어요. 노드 전체에 SG를 여는 것보다 훨씬 안전해요.

---

## 🦥 정리

| 주제 | 로컬 K8s | EKS |
|------|---------|-----|
| **LoadBalancer** | MetalLB 필요 | AWS NLB/CLB 자동 생성 |
| **Ingress** | Nginx Ingress Controller | ALB Ingress (AWS LB Controller) |
| **SSL/TLS** | cert-manager + Let's Encrypt | ACM (무료, 자동 갱신) |
| **내부 DNS** | CoreDNS (동일) | CoreDNS (동일) |
| **외부 DNS** | 수동 관리 | ExternalDNS → Route53 자동 |
| **네트워크 보안** | NetworkPolicy | NetworkPolicy + SG for Pods |

다음 포스트에서는 **스토리지 비교 — hostPath/NFS vs EBS/EFS CSI**를 다룰게요! 🦥
