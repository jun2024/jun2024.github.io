---
title: "[Kubernetes] K8s Networking (5) — Ingress 완벽 가이드"
excerpt: "K8s 외부 트래픽 관리의 핵심! Ingress Controller, Ingress Resource, Path/Host 기반 라우팅, Annotations & rewrite-target까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Networking, Ingress, Nginx, CKA, DevOps]

permalink: /kubernetes/kubernetes-networking-5/

toc: true
toc_sticky: true

date: 2026-06-08
last_modified_at: 2026-06-08
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Networking 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [Networking (4)](/kubernetes/kubernetes-networking-4/)에서 Service Networking, DNS & CoreDNS를 다뤘어요!

---

## 🦥 왜 Ingress가 필요할까?

### Service만으로는 부족한 이유

K8s Service로 애플리케이션을 외부에 노출할 수 있어요. 하지만 실제 프로덕션에서는 한계가 있어요.

| Service Type | 한계 |
|-------------|------|
| **NodePort** | 30000-32767 범위의 포트만 사용, URL에 포트 번호 포함 |
| **LoadBalancer** | 클라우드 LB 1개당 Service 1개 = 비용 증가 |

예를 들어 쇼핑몰을 운영한다고 해볼게요.

| 서비스 | 필요한 LoadBalancer |
|--------|-------------------|
| 메인 사이트 (www) | LB 1 |
| 상품 API (/api) | LB 2 |
| 결제 서비스 (/pay) | LB 3 |

서비스가 3개면 클라우드 LB도 3개 필요해요. 비용도 3배고, SSL 인증서도 각각 관리해야 해요.

### Ingress의 역할

**Ingress**는 클러스터 외부에서 들어오는 HTTP/HTTPS 트래픽을 **URL 경로나 호스트 이름 기반으로** 내부 Service에 라우팅해요. 하나의 진입점으로 여러 Service를 관리할 수 있어요.

| Ingress 기능 | 설명 |
|-------------|------|
| URL 기반 라우팅 | `/api` → api-service, `/pay` → pay-service |
| 호스트 기반 라우팅 | `api.example.com` → api-svc, `web.example.com` → web-svc |
| SSL/TLS 종료 | Ingress에서 HTTPS 처리 |
| 로드밸런싱 | 여러 Pod에 트래픽 분산 |

---

## 🦥 Ingress Controller vs Ingress Resource

Ingress는 두 부분으로 나뉘어요. 헷갈리기 쉬우니 확실히 구분해야 해요.

| 구분 | Ingress Controller | Ingress Resource |
|------|-------------------|-----------------|
| 역할 | 실제 트래픽 처리 (Reverse Proxy) | 라우팅 규칙 정의 |
| 비유 | Nginx 서버 자체 | Nginx 설정 파일 |
| 배포 | Deployment로 설치 | YAML로 규칙 작성 |
| 기본 제공 | K8s에 기본 포함 **아님** | K8s API 리소스 |

> 💡 **CKA 핵심**: K8s에는 Ingress Controller가 기본으로 설치되어 있지 않아요. 직접 배포해야 해요!

### 주요 Ingress Controller

| Controller | 특징 |
|-----------|------|
| **Nginx Ingress Controller** | 가장 널리 사용, K8s 공식 지원 |
| **HAProxy** | 고성능 로드밸런서 |
| **Traefik** | 자동 설정, Let's Encrypt 통합 |
| **Istio Gateway** | 서비스 메시와 통합 |
| **GKE Ingress** | GCP 전용 (GCE LB 사용) |

---

## 🦥 Nginx Ingress Controller 배포

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
    spec:
      containers:
      - name: controller
        image: registry.k8s.io/ingress-nginx/controller:v1.9.0
        args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

### 함께 필요한 리소스

| 리소스 | 용도 |
|--------|------|
| **ConfigMap** | Nginx 설정 커스터마이징 |
| **Service** (NodePort/LoadBalancer) | 외부에서 Controller 접근 |
| **ServiceAccount** | 권한 관리 (RBAC) |
| **Role/ClusterRole** | Ingress, Service, Secret 등 조회 권한 |

```bash
# 한 번에 설치 (공식 매니페스트)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/baremetal/deploy.yaml

# 설치 확인
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## 🦥 Ingress Resource — 라우팅 규칙

### 단일 Service 라우팅 (Default Backend)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  defaultBackend:
    service:
      name: my-service
      port:
        number: 80
```

모든 트래픽을 하나의 Service로 보내요.

### Path 기반 라우팅

URL 경로에 따라 다른 Service로 라우팅해요.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

### pathType 종류

| pathType | 매칭 방식 | `/wear` 규칙에 매칭되는 URL |
|----------|----------|--------------------------|
| `Prefix` | 접두사 매칭 | `/wear`, `/wear/`, `/wear/shirt` |
| `Exact` | 정확히 일치 | `/wear`만 매칭 |

### Host 기반 라우팅

도메인 이름에 따라 다른 Service로 라우팅해요.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  rules:
  - host: wear.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
  - host: watch.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

### Path + Host 조합

```yaml
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
```

---

## 🦥 Annotations & Rewrite Target

### Rewrite가 필요한 이유

Ingress에서 `/wear`로 들어온 요청이 `wear-service`로 전달될 때, 기본적으로 **경로가 그대로 전달**돼요.

| 사용자 요청 | Service가 받는 요청 | 문제 |
|-----------|-------------------|------|
| `example.com/wear` | `/wear` | wear-service는 `/` 경로를 기대 |
| `example.com/wear/shirt` | `/wear/shirt` | wear-service에 `/wear/shirt` 경로 없음 → 404 |

### rewrite-target Annotation

`rewrite-target`을 사용하면 경로를 변환해서 전달할 수 있어요.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

| 사용자 요청 | rewrite 후 Service가 받는 요청 |
|-----------|-------------------------------|
| `example.com/wear` | `/` |
| `example.com/wear/shirt` | `/` |

### Regex를 사용한 Rewrite

더 정밀한 경로 변환이 필요할 때 정규표현식을 사용해요.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
  - http:
      paths:
      - path: /wear(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: wear-service
            port:
              number: 80
```

| 사용자 요청 | `$2` 캡처 | Service가 받는 요청 |
|-----------|----------|-------------------|
| `/wear` | (빈 문자열) | `/` |
| `/wear/` | (빈 문자열) | `/` |
| `/wear/shirt` | `shirt` | `/shirt` |
| `/wear/shirt/blue` | `shirt/blue` | `/shirt/blue` |

### 자주 사용하는 Annotations

| Annotation | 설명 |
|-----------|------|
| `nginx.ingress.kubernetes.io/rewrite-target` | URL 경로 재작성 |
| `nginx.ingress.kubernetes.io/ssl-redirect` | HTTPS 리다이렉트 |
| `nginx.ingress.kubernetes.io/use-regex` | 정규표현식 사용 |
| `nginx.ingress.kubernetes.io/proxy-body-size` | 요청 본문 크기 제한 |
| `nginx.ingress.kubernetes.io/affinity` | 세션 어피니티 |

---

## 🦥 Ingress 관리 명령어

```bash
# Ingress 목록 확인
kubectl get ingress
kubectl get ing          # 약어

# Ingress 상세 정보
kubectl describe ingress my-ingress

# Ingress 생성 (명령형)
kubectl create ingress my-ingress \
  --rule="example.com/wear=wear-svc:80" \
  --rule="example.com/watch=watch-svc:80"

# Ingress YAML 생성 (dry-run)
kubectl create ingress my-ingress \
  --rule="example.com/wear=wear-svc:80" \
  --dry-run=client -o yaml > ingress.yaml
```

---

## 🦥 CKA 시험 대비 — Ingress 치트시트

```bash
# Ingress Controller 확인
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Ingress Resource 확인
kubectl get ingress -A
kubectl describe ingress <name>

# 빠른 Ingress 생성
kubectl create ingress test \
  --rule="/path=service:port" \
  --annotation nginx.ingress.kubernetes.io/rewrite-target=/

# Ingress Controller 로그 확인
kubectl logs -n ingress-nginx <controller-pod>
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Ingress 필요성** | NodePort 포트 제한, LB 비용 → Ingress로 통합 |
| **Controller vs Resource** | Controller = Nginx 서버, Resource = 설정 파일 |
| **Ingress Controller** | K8s에 기본 미포함, 직접 설치 필요 (Nginx 등) |
| **Path 라우팅** | URL 경로 기반 (`/wear`, `/watch`) |
| **Host 라우팅** | 도메인 기반 (`api.example.com`, `web.example.com`) |
| **pathType** | Prefix (접두사), Exact (정확 매칭) |
| **rewrite-target** | 경로 변환 (Ingress 경로 → Service 경로) |
| **Regex rewrite** | `/$2`로 세밀한 경로 매핑 |

이것으로 Networking 섹션을 마무리할게요! Linux 네트워크 기초부터 K8s의 Pod/Service 네트워킹, DNS, 그리고 Ingress까지 — CKA 시험에서 가장 비중이 큰 섹션을 다뤘어요. 🦥
