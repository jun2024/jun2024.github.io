---
title: "[Kubernetes] K8s 보안 완벽 정리 (3) — KubeConfig, API Groups, Authorization, RBAC"
excerpt: "kubeconfig 파일 구조와 컨텍스트 전환, K8s API 그룹 체계, 그리고 RBAC를 활용한 권한 관리까지 — CKA 인가(Authorization) 핵심!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Security, RBAC, KubeConfig, CKA, DevOps]

permalink: /kubernetes/kubernetes-security-3/

toc: true
toc_sticky: true

date: 2026-05-11
last_modified_at: 2026-05-11
---

> 이전 포스트 [K8s 보안 (2)](/kubernetes/kubernetes-security-2/)에서 TLS 구조와 인증서 관리를 다뤘어요.
> 이번에는 KubeConfig와 RBAC 기반 권한 관리를 알아볼게요!

---

## 🦥 KubeConfig

### 왜 KubeConfig가 필요한가?

매번 kubectl 명령어에 인증서 경로를 지정하는 건 너무 번거로워요.

```bash
# 매번 이렇게 입력해야 한다면...
kubectl get pods \
  --server=https://kube-apiserver:6443 \
  --client-key=admin.key \
  --client-certificate=admin.crt \
  --certificate-authority=ca.crt
```

KubeConfig 파일에 이 정보를 저장해두면 자동으로 사용돼요. 기본 경로는 `~/.kube/config`예요.

### KubeConfig 구조

KubeConfig는 3가지 섹션으로 구성돼요.

| 섹션 | 내용 | 예시 |
|------|------|------|
| **clusters** | 접속할 클러스터 정보 (서버 URL, CA 인증서) | production, staging |
| **users** | 인증 정보 (클라이언트 인증서, 키) | admin, developer |
| **contexts** | cluster + user 조합 | admin@production |

```yaml
apiVersion: v1
kind: Config
current-context: admin@production

clusters:
- name: production
  cluster:
    server: https://production-server:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt
- name: staging
  cluster:
    server: https://staging-server:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt

users:
- name: admin
  user:
    client-certificate: /etc/kubernetes/pki/admin.crt
    client-key: /etc/kubernetes/pki/admin.key
- name: developer
  user:
    client-certificate: /etc/kubernetes/pki/developer.crt
    client-key: /etc/kubernetes/pki/developer.key

contexts:
- name: admin@production
  context:
    cluster: production
    user: admin
    namespace: default
- name: developer@staging
  context:
    cluster: staging
    user: developer
    namespace: dev
```

### KubeConfig 관리 명령어

```bash
# 현재 설정 확인
kubectl config view

# 현재 컨텍스트 확인
kubectl config current-context

# 컨텍스트 전환
kubectl config use-context developer@staging

# 특정 kubeconfig 파일 사용
kubectl config view --kubeconfig=/path/to/custom-config

# 네임스페이스 변경
kubectl config set-context --current --namespace=dev
```

> 💡 **CKA 팁**: 시험에서 여러 클러스터를 오가는 문제가 나와요. `kubectl config use-context`로 빠르게 전환하는 연습을 하세요!

### 인증서를 Base64로 내장하기

파일 경로 대신 인증서 내용을 직접 넣을 수도 있어요.

```yaml
clusters:
- name: production
  cluster:
    server: https://production-server:6443
    certificate-authority-data: LS0tLS1CRUdJ...   # base64 인코딩된 CA 인증서

users:
- name: admin
  user:
    client-certificate-data: LS0tLS1CRUdJ...      # base64 인코딩된 인증서
    client-key-data: LS0tLS1CRUdJ...               # base64 인코딩된 키
```

---

## 🦥 API Groups

### K8s API 구조

K8s API는 기능별로 **그룹**으로 나뉘어져 있어요.

| API 그룹 | 경로 | 리소스 예시 |
|----------|------|-----------|
| **core (v1)** | `/api/v1` | pods, services, configmaps, secrets, nodes, namespaces |
| **apps** | `/apis/apps/v1` | deployments, replicasets, statefulsets, daemonsets |
| **networking.k8s.io** | `/apis/networking.k8s.io/v1` | ingresses, networkpolicies |
| **rbac.authorization.k8s.io** | `/apis/rbac.authorization.k8s.io/v1` | roles, rolebindings, clusterroles |
| **certificates.k8s.io** | `/apis/certificates.k8s.io/v1` | certificatesigningrequests |
| **storage.k8s.io** | `/apis/storage.k8s.io/v1` | storageclasses |

```bash
# API 그룹 목록 확인
kubectl api-resources

# 특정 리소스의 API 그룹 확인
kubectl api-resources | grep deployment
# deployments   deploy   apps/v1   true   Deployment
```

> 💡 RBAC에서 Role을 만들 때 `apiGroups` 필드에 정확한 API 그룹을 지정해야 해요. core 그룹은 `""`(빈 문자열)로 표현해요.

---

## 🦥 Authorization — 무엇을 할 수 있는가?

### Authorization 모드

| 모드 | 설명 | 사용 시기 |
|------|------|----------|
| **Node** | kubelet의 API 접근 권한 자동 부여 | 모든 클러스터 (자동) |
| **ABAC** | 속성 기반 접근 제어, JSON 파일로 관리 | 거의 사용 안 함 |
| **RBAC** | 역할 기반 접근 제어, K8s 리소스로 관리 | **가장 많이 사용** |
| **Webhook** | 외부 서비스에 인가 판단 위임 | Open Policy Agent 등 |
| **AlwaysAllow** | 모든 요청 허용 (기본값) | 테스트 환경 |
| **AlwaysDeny** | 모든 요청 거부 | 사용 안 함 |

```bash
# 현재 Authorization 모드 확인
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
# --authorization-mode=Node,RBAC
```

> 💡 여러 모드를 지정하면 **순서대로 체크**해요. 한 모드에서 허용되면 다음 모드는 체크하지 않아요.

---

## 🦥 RBAC (Role-Based Access Control)

### RBAC 리소스 구조

| 리소스 | 범위 | 역할 |
|--------|------|------|
| **Role** | 네임스페이스 | 특정 네임스페이스 내 권한 정의 |
| **RoleBinding** | 네임스페이스 | Role을 사용자/그룹에 연결 |
| **ClusterRole** | 클러스터 전체 | 클러스터 전체 또는 비 네임스페이스 리소스 권한 |
| **ClusterRoleBinding** | 클러스터 전체 | ClusterRole을 사용자/그룹에 연결 |

### Role 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: default
rules:
- apiGroups: [""]              # core 그룹 (pods, services 등)
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]          # apps 그룹 (deployments 등)
  resources: ["deployments"]
  verbs: ["get", "list", "create"]
```

### RoleBinding 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### 명령형(Imperative)으로 생성

```bash
# Role 생성
kubectl create role developer-role \
  --verb=get,list,create,update,delete \
  --resource=pods \
  --namespace=default

# RoleBinding 생성
kubectl create rolebinding developer-binding \
  --role=developer-role \
  --user=jane \
  --namespace=default
```

### 권한 확인 (can-i)

```bash
# 내 권한 확인
kubectl auth can-i create deployments
kubectl auth can-i delete pods --namespace=dev

# 다른 사용자 권한 확인 (관리자만)
kubectl auth can-i create pods --as=jane
kubectl auth can-i delete nodes --as=jane
kubectl auth can-i get pods --as=jane --namespace=dev
```

### 주요 verbs 목록

| Verb | 설명 | kubectl 명령어 |
|------|------|---------------|
| `get` | 리소스 조회 | `kubectl get`, `kubectl describe` |
| `list` | 리소스 목록 조회 | `kubectl get` (목록) |
| `watch` | 리소스 변경 감시 | `kubectl get -w` |
| `create` | 리소스 생성 | `kubectl create`, `kubectl apply` |
| `update` | 리소스 수정 | `kubectl edit`, `kubectl apply` |
| `patch` | 리소스 부분 수정 | `kubectl patch` |
| `delete` | 리소스 삭제 | `kubectl delete` |

### 특정 리소스 이름으로 제한

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  resourceNames: ["web-app", "api-server"]   # 특정 Pod에만 권한 부여
  verbs: ["get", "watch"]
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **KubeConfig** | clusters + users + contexts, `~/.kube/config`에 저장 |
| **컨텍스트 전환** | `kubectl config use-context`, `--namespace` 설정 |
| **API Groups** | core = `""`, apps, networking 등 기능별 그룹 |
| **Authorization** | Node, RBAC, Webhook, ABAC — RBAC가 표준 |
| **Role/RoleBinding** | 네임스페이스 범위 권한, `verbs` + `resources` + `apiGroups` |
| **can-i** | `kubectl auth can-i`로 권한 확인, `--as`로 다른 사용자 확인 |

다음 포스트에서는 **ClusterRole, Image Security, SecurityContext, NetworkPolicy**를 다뤄볼게요! 🦥
