---
title: "[Kubernetes] K8s 네트워크 핵심 개념 완벽 정리"
excerpt: "Pod 네트워킹부터 Service, Ingress, NetworkPolicy, 서비스 메시까지 — K8s 네트워크의 모든 것을 예시 코드와 함께 정리해보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Network, Ingress, Istio, Service Mesh, DevOps]

permalink: /kubernetes/kubernetes-network/

toc: true
toc_sticky: true

date: 2026-04-01
last_modified_at: 2026-04-01
---

## 🦥 K8s 네트워크 기본 원칙

K8s 네트워크에는 세 가지 핵심 원칙이 있어요.

> 1. **모든 Pod은 NAT 없이 서로 직접 통신**할 수 있어야 한다.
> 2. **모든 Node는 NAT 없이 모든 Pod과 통신**할 수 있어야 한다.
> 3. **Pod이 자신이 인식하는 IP와 다른 Pod이 보는 IP가 같아야** 한다.

이 원칙들을 구현하기 위해 CNI 플러그인, Service, Ingress, NetworkPolicy 등 다양한 네트워크 컴포넌트가 존재해요. 하나씩 살펴볼게요!

---

## 🦥 CNI (Container Network Interface)

**CNI**는 K8s의 Pod 네트워킹을 담당하는 플러그인 인터페이스예요. 각 Pod에 고유한 IP를 부여하고, Pod 간 통신 경로를 만들어줘요.

### 대표적인 CNI 플러그인

| CNI | 특징 |
|---|---|
| `Calico` | BGP 기반 라우팅, NetworkPolicy 지원, 가장 널리 사용됨 |
| `Cilium` | eBPF 기반, 고성능, L7 정책 지원 |
| `Flannel` | 심플한 오버레이 네트워크, 소규모 클러스터에 적합 |
| `WeaveNet` | 메시 네트워크, 설치가 간편 |

### Calico 설치 예시

```bash
# Calico CNI 설치 (Operator 방식)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

# 설치 확인
kubectl get pods -n calico-system
```

### Cilium 설치 예시

```bash
# Cilium CLI로 설치
cilium install --version 1.15.0

# 상태 확인
cilium status
```

---

## 🦥 Service

Pod은 언제든 죽고 다시 생길 수 있어서 IP가 수시로 바뀌어요. **Service**는 Pod 집합에 안정적인 가상 IP(ClusterIP)와 DNS 이름을 제공해서 이 문제를 해결해줘요.

### Service 타입별 정리

| 타입 | 접근 범위 | 용도 |
|---|---|---|
| `ClusterIP` | 클러스터 내부만 | 내부 마이크로서비스 간 통신 (기본값) |
| `NodePort` | 외부에서 노드IP:포트로 | 개발/테스트 환경 |
| `LoadBalancer` | 외부 LB를 통해 | 프로덕션 외부 노출 |
| `ExternalName` | 외부 DNS로 리다이렉트 | 외부 서비스 연동 |

### ClusterIP Service 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # 기본값이라 생략 가능
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80        # Service가 노출하는 포트
      targetPort: 8080 # Pod이 실제로 리스닝하는 포트
```

### NodePort Service 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
      nodePort: 30080  # 30000-32767 범위, 생략하면 자동 할당
```

### LoadBalancer Service 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-loadbalancer
  annotations:
    # AWS NLB를 사용하고 싶을 때
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: api-server
  ports:
    - protocol: TCP
      port: 443
      targetPort: 8443
```

> **주의!** LoadBalancer 타입은 서비스 하나당 LB 하나가 생겨요. 서비스가 10개면 LB도 10개 — 비용이 빠르게 늘어나죠. 이걸 해결하는 게 바로 **Ingress**예요!

---

## 🦥 kube-proxy

**kube-proxy**는 각 노드에서 실행되면서 Service의 가상 IP를 실제 Pod IP로 변환해주는 컴포넌트예요.

### 동작 모드

| 모드 | 특징 |
|---|---|
| `iptables` | 기본 모드, 규칙 기반 패킷 필터링 |
| `IPVS` | 대규모 클러스터에 유리, 해시 테이블 기반으로 성능 좋음 |
| `nftables` | K8s 1.29+, iptables의 후속 |

### IPVS 모드로 변경 예시

```yaml
# kube-proxy ConfigMap 수정
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: "ipvs"
    ipvs:
      scheduler: "rr"  # round-robin, lc, sh 등 선택 가능
```

```bash
# 변경 후 kube-proxy 재시작
kubectl rollout restart daemonset kube-proxy -n kube-system

# IPVS 규칙 확인
kubectl exec -n kube-system kube-proxy-xxxxx -- ipvsadm -Ln
```

---

## 🦥 CoreDNS (클러스터 내부 DNS)

**CoreDNS**는 클러스터 내부의 DNS 서버예요. Service 이름으로 Pod에 접근할 수 있게 해줘요.

### DNS 해석 규칙

```
<service-name>.<namespace>.svc.cluster.local
```

같은 네임스페이스에서는 서비스 이름만으로 접근 가능해요!

```bash
# 같은 네임스페이스
curl http://backend-service

# 다른 네임스페이스
curl http://backend-service.production.svc.cluster.local
```

### CoreDNS 커스텀 설정 예시

```yaml
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
        # 사내 DNS로 포워딩
        forward . 10.0.0.2 {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

---

## 🦥 Ingress

**Ingress**는 하나의 진입점으로 여러 서비스에 L7(HTTP/HTTPS) 라우팅을 해주는 리소스예요. 호스트명이나 URL 경로 기반으로 트래픽을 분배할 수 있어요.

### Ingress의 핵심 포인트

- Ingress **리소스** = 라우팅 규칙 정의 (YAML)
- Ingress **Controller** = 실제 구현체 (Nginx, Traefik, AWS ALB Controller 등)
- Controller 없이 Ingress 리소스만 만들면 아무 일도 안 일어나요!

### Nginx Ingress Controller 설치

```bash
# Helm으로 설치
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Ingress 리소스 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: tls-secret
  rules:
    # 호스트 기반 라우팅
    - host: myapp.example.com
      http:
        paths:
          # 경로 기반 라우팅
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

---

## 🦥 Gateway API

**Gateway API**는 Ingress의 후속 표준으로, K8s 커뮤니티가 공식 추진하고 있는 차세대 라우팅 규격이에요.

### Ingress vs Gateway API

| 비교 | Ingress | Gateway API |
|---|---|---|
| 프로토콜 | HTTP/HTTPS만 | HTTP, gRPC, TCP, UDP 모두 지원 |
| 역할 분리 | 없음 | 인프라팀(Gateway) / 개발팀(Route) 분리 |
| 표현력 | 제한적 (annotation 남발) | 풍부한 네이티브 옵션 |
| 상태 | 안정 (GA) | 점점 GA 진행 중 |

### Gateway API 구성 예시

```yaml
# 1. GatewayClass — 인프라 관리자가 정의
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: istio
spec:
  controllerName: istio.io/gateway-controller
---
# 2. Gateway — 플랫폼팀이 정의
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: infra
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-cert
---
# 3. HTTPRoute — 개발팀이 정의
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: app-team
spec:
  parentRefs:
    - name: main-gateway
      namespace: infra
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - name: api-v1
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /v2
      backendRefs:
        - name: api-v2
          port: 80
```

역할이 깔끔하게 분리되니까 대규모 팀에서 특히 유용해요!

---

## 🦥 NetworkPolicy

**NetworkPolicy**는 Pod 간 트래픽을 방화벽처럼 제어하는 리소스예요. 기본적으로 K8s는 모든 Pod 간 통신을 허용하는데, NetworkPolicy를 정의하면 화이트리스트 방식으로 제어할 수 있어요.

> **주의!** CNI 플러그인이 NetworkPolicy를 지원해야 동작해요. Calico, Cilium은 지원하지만 **Flannel은 미지원**이에요.

### 모든 인바운드 트래픽 차단 (기본 정책)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}  # 네임스페이스 내 모든 Pod에 적용
  policyTypes:
    - Ingress
  # ingress 규칙이 없으므로 모든 인바운드 차단
```

### 특정 Pod에서만 접근 허용

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend  # 이 정책이 적용되는 대상
  policyTypes:
    - Ingress
  ingress:
    - from:
        # app=frontend 라벨을 가진 Pod에서만 허용
        - podSelector:
            matchLabels:
              app: frontend
        # 특정 네임스페이스에서만 허용
        - namespaceSelector:
            matchLabels:
              env: staging
      ports:
        - protocol: TCP
          port: 8080
```

### Egress(아웃바운드) 제어

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    # DNS 허용 (필수!)
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # 특정 외부 IP 대역만 허용
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 5432  # PostgreSQL
```

---

## 🦥 서비스 메시 (Service Mesh)

서비스 메시는 마이크로서비스 간 통신을 **인프라 레벨에서 제어**하는 전용 레이어예요. 앞서 살펴본 Ingress나 Service가 외부→내부(**North-South**) 트래픽에 집중한다면, 서비스 메시는 서비스 간 내부 통신(**East-West**)까지 모두 관리해줘요.

### Ingress vs Istio vs 외부 LB 비교

먼저 이 세 가지가 어떤 관계인지 한눈에 정리해볼게요.

```
[Client]
   │
   ▼
[외부 LB] ─── 인프라 레벨, L4/L7 분산
   │
   ▼
[Ingress Controller / Istio Gateway] ─── L7 라우팅 규칙
   │
   ▼
[Service → Pod] ─── Istio는 여기 Pod 간 통신도 제어
```

| 항목 | 외부 LB (AWS ELB 등) | Ingress | Istio (서비스 메시) |
|---|---|---|---|
| **동작 레이어** | L4/L7 | L7 | L4~L7 |
| **위치** | 클러스터 바깥 (클라우드) | 클러스터 안 (Controller Pod) | 클러스터 안 (사이드카) |
| **트래픽 방향** | North-South | North-South | North-South + East-West |
| **주요 기능** | 단순 부하 분산, 헬스체크 | 호스트/경로 라우팅, TLS 종료 | mTLS, 트래픽 정책, 관측성, 카나리 배포 |
| **서비스당 비용** | LB 1개 = 비용 1개 | 하나의 LB로 여러 서비스 라우팅 | 사이드카 프록시 리소스 |
| **복잡도** | 낮음 | 중간 | 높음 |
| **적합한 시나리오** | 단순 외부 노출 | HTTP 기반 다중 서비스 라우팅 | 대규모 마이크로서비스 아키텍처 |

---

### 주요 서비스 메시 솔루션

#### 1. Istio

가장 널리 사용되는 서비스 메시예요. 각 Pod에 **Envoy** 사이드카 프록시를 자동 주입해서 모든 트래픽을 가로채고 제어해요.

- **mTLS 자동 암호화**: 서비스 간 통신을 자동으로 TLS 암호화
- **트래픽 관리**: 카나리 배포, A/B 테스트, 서킷 브레이커, 재시도/타임아웃
- **관측성**: 분산 트레이싱(Jaeger), 메트릭(Prometheus), 로깅
- **최근 동향**: 사이드카 없는 **Ambient Mesh** 모드 등장 (ztunnel 기반)

```bash
# Istio 설치
istioctl install --set profile=demo -y

# 네임스페이스에 사이드카 자동 주입 활성화
kubectl label namespace default istio-injection=enabled
```

#### 2. Linkerd

**경량화**에 초점을 맞춘 서비스 메시예요. Rust로 작성된 자체 프록시(**linkerd2-proxy**)를 사용해서 리소스 오버헤드가 Istio보다 훨씬 작아요.

- **초경량 프록시**: Envoy 대비 메모리/CPU 사용량 현저히 낮음
- **간편한 설치**: 몇 분 만에 설치 완료
- **자동 mTLS**: 제로 컨피그로 서비스 간 암호화
- **적합한 경우**: 복잡한 트래픽 제어보다 보안과 관측성이 우선일 때

```bash
# Linkerd CLI 설치 후
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# 상태 확인
linkerd check
```

#### 3. Cilium Service Mesh

**eBPF** 기반으로 동작하는 차세대 서비스 메시예요. 사이드카 없이 커널 레벨에서 네트워킹을 처리해서 성능이 뛰어나요.

- **사이드카리스(Sidecar-less)**: 사이드카 프록시 없이 eBPF로 직접 처리
- **높은 성능**: 커널 레벨 처리로 지연 시간 최소화
- **CNI + 서비스 메시 통합**: Cilium CNI 위에 서비스 메시 기능이 내장
- **L7 정책**: HTTP, gRPC, Kafka 등 프로토콜 인식 정책

```bash
# Cilium을 서비스 메시 모드로 설치
cilium install --version 1.15.0 \
  --set kubeProxyReplacement=true \
  --set envoy.enabled=true

# 서비스 메시 기능 활성화
cilium hubble enable --ui
```

#### 4. Consul Connect (HashiCorp)

HashiCorp의 **Consul**이 제공하는 서비스 메시 기능이에요. K8s 외에도 VM, 베어메탈 등 **멀티 플랫폼**을 지원하는 게 큰 강점이에요.

- **멀티 플랫폼**: K8s + VM + 베어메탈 혼합 환경 지원
- **서비스 디스커버리 내장**: Consul 자체가 서비스 레지스트리
- **Intentions 기반 접근 제어**: 직관적인 서비스 간 접근 정책
- **멀티 데이터센터**: 여러 데이터센터/클러스터 간 메시 연결

```bash
# Helm으로 Consul 설치
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install consul hashicorp/consul \
  --namespace consul \
  --create-namespace \
  --set global.name=consul \
  --set connectInject.enabled=true
```

### 서비스 메시 비교 요약

| 항목 | Istio | Linkerd | Cilium | Consul Connect |
|---|---|---|---|---|
| **프록시** | Envoy (사이드카) | linkerd2-proxy (사이드카) | eBPF (사이드카리스) | Envoy (사이드카) |
| **리소스 오버헤드** | 높음 | 낮음 | 매우 낮음 | 중간 |
| **기능 풍부도** | 매우 높음 | 중간 | 높음 (성장 중) | 높음 |
| **학습 곡선** | 가파름 | 완만함 | 중간 | 중간 |
| **멀티 플랫폼** | K8s 중심 | K8s 전용 | K8s 전용 | K8s + VM + 베어메탈 |
| **적합한 상황** | 대규모 마이크로서비스 | 빠른 도입, 경량 운영 | 고성능 요구 환경 | 하이브리드 인프라 |

> 서비스 메시 각 솔루션의 심화 내용과 실전 구성은 다음 포스트에서 더 깊게 다뤄볼게요!

---

## 🦥 전체 트래픽 흐름 요약

마지막으로 K8s 네트워크의 전체 트래픽 흐름을 정리해볼게요.

```
외부 클라이언트
  → 외부 LB (AWS ELB/NLB — L4 부하 분산)
    → Ingress Controller 또는 Istio Gateway (L7 라우팅)
      → Service (kube-proxy가 ClusterIP → Pod IP 변환)
        → Pod (서비스 메시 사용 시 사이드카 프록시가 가로채서 처리)
          → 다른 Service의 Pod (East-West 통신, mTLS 적용)
```

K8s 네트워크는 겉보기엔 복잡하지만, **CNI → Service → Ingress → 서비스 메시** 순서로 레이어를 쌓아간다고 이해하면 훨씬 명확해져요. 각 레이어가 해결하는 문제가 다르고, 프로젝트 규모와 요구사항에 맞게 선택하면 돼요!
