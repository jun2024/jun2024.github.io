---
title: "[Kubernetes] K8s 보안 완벽 정리 (4) — ClusterRole, Image Security, SecurityContext, NetworkPolicy"
excerpt: "클러스터 전체 권한 관리(ClusterRole), 프라이빗 레지스트리 인증, Pod 보안 컨텍스트, 네트워크 정책까지 — CKA 보안 마무리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Security, ClusterRole, NetworkPolicy, CKA, DevOps]

permalink: /kubernetes/kubernetes-security-4/

toc: true
toc_sticky: true

date: 2026-05-04
last_modified_at: 2026-05-04
---

> 이전 포스트 [K8s 보안 (3)](/kubernetes/kubernetes-security-3/)에서 KubeConfig, API Groups, RBAC를 다뤘어요.
> 이번 포스트로 K8s 보안 시리즈를 마무리해요!

---

## 🦥 ClusterRole & ClusterRoleBinding

### Role vs ClusterRole

Role은 **특정 네임스페이스** 내에서만 유효하지만, ClusterRole은 **클러스터 전체**에 적용돼요.

| 항목 | Role / RoleBinding | ClusterRole / ClusterRoleBinding |
|------|-------------------|--------------------------------|
| **범위** | 네임스페이스 내 | 클러스터 전체 |
| **대상 리소스** | Pods, Deployments, Services 등 | Nodes, PV, Namespaces 등 **비 네임스페이스 리소스** 포함 |
| **사용 시기** | 특정 네임스페이스 권한 | 클러스터 관리, 비 네임스페이스 리소스 접근 |

### 네임스페이스 리소스 vs 비 네임스페이스 리소스

```bash
# 네임스페이스 리소스 목록
kubectl api-resources --namespaced=true
# pods, deployments, services, configmaps, secrets, roles, rolebindings...

# 비 네임스페이스 리소스 목록
kubectl api-resources --namespaced=false
# nodes, persistentvolumes, namespaces, clusterroles, clusterrolebindings...
```

### ClusterRole 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

### ClusterRoleBinding 생성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### 명령형으로 생성

```bash
kubectl create clusterrole node-reader \
  --verb=get,list,watch --resource=nodes

kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader --user=jane
```

> 💡 ClusterRole을 RoleBinding으로 바인딩하면 **해당 네임스페이스에서만** ClusterRole의 권한이 적용돼요. 여러 네임스페이스에 같은 권한을 주고 싶을 때 유용해요.

---

## 🦥 Image Security

### 이미지 이름의 전체 구조

```
docker.io/library/nginx:1.25
│          │       │     │
│          │       │     └── Tag
│          │       └── Image Name
│          └── User/Organization (library = 공식)
└── Registry (기본: docker.io)
```

| 예시 | Registry | 실제 전체 경로 |
|------|----------|--------------|
| `nginx` | docker.io | `docker.io/library/nginx:latest` |
| `myorg/myapp` | docker.io | `docker.io/myorg/myapp:latest` |
| `gcr.io/google-containers/busybox` | gcr.io | 그대로 |

### Private Registry 사용

프라이빗 레지스트리의 이미지를 사용하려면 인증이 필요해요.

**Step 1: docker-registry Secret 생성**

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com
```

**Step 2: Pod에서 imagePullSecrets 지정**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
  - name: app
    image: private-registry.io/myorg/myapp:1.0
  imagePullSecrets:
  - name: regcred
```

> 💡 **CKA 팁**: `kubectl create secret docker-registry` 명령어를 기억하세요. Secret 타입이 `kubernetes.io/dockerconfigjson`으로 자동 설정돼요.

---

## 🦥 Security Context

### Security Context란?

Security Context는 Pod 또는 컨테이너의 **보안 관련 설정**을 정의해요. 실행 사용자, 권한, Linux Capabilities 등을 제어할 수 있어요.

### Pod 레벨 vs 컨테이너 레벨

| 레벨 | 적용 범위 | 설정 위치 |
|------|----------|----------|
| **Pod 레벨** | Pod 내 모든 컨테이너에 적용 | `spec.securityContext` |
| **컨테이너 레벨** | 해당 컨테이너에만 적용 | `spec.containers[].securityContext` |

> 💡 컨테이너 레벨 설정이 Pod 레벨보다 **우선 적용**돼요.

### 실행 사용자 설정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-demo
spec:
  # Pod 레벨: 전체 컨테이너에 적용
  securityContext:
    runAsUser: 1000        # UID 1000으로 실행
    runAsGroup: 3000       # GID 3000
    fsGroup: 2000          # Volume 마운트 시 그룹

  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]

    # 컨테이너 레벨: 이 컨테이너에만 적용
    securityContext:
      runAsUser: 2000      # Pod 레벨의 1000을 덮어씀
      allowPrivilegeEscalation: false
```

### Linux Capabilities

기본적으로 컨테이너는 root로 실행되어도 모든 Linux Capabilities를 갖지 않아요. 필요한 Capability만 추가하거나, 불필요한 것을 제거할 수 있어요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cap-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]    # Capability 추가
        drop: ["ALL"]                       # 모든 Capability 제거 후 필요한 것만 추가
```

> 💡 **CKA 포인트**: Capabilities는 **컨테이너 레벨**에서만 설정 가능해요. Pod 레벨에서는 설정할 수 없어요!

### 주요 securityContext 필드

| 필드 | 레벨 | 설명 |
|------|------|------|
| `runAsUser` | Pod/Container | 실행 UID |
| `runAsGroup` | Pod/Container | 실행 GID |
| `runAsNonRoot` | Pod/Container | root 실행 차단 (true/false) |
| `fsGroup` | Pod | Volume 파일 소유 그룹 |
| `readOnlyRootFilesystem` | Container | 루트 파일시스템 읽기 전용 |
| `allowPrivilegeEscalation` | Container | 권한 상승 차단 |
| `capabilities` | Container | Linux Capabilities 추가/제거 |

---

## 🦥 Network Policies

### Network Policy란?

기본적으로 K8s의 모든 Pod은 **모든 다른 Pod과 통신 가능**해요. Network Policy를 사용하면 특정 Pod의 Ingress(들어오는)/Egress(나가는) 트래픽을 제한할 수 있어요.

### 기본 동작

| Network Policy 없을 때 | Network Policy 있을 때 |
|----------------------|---------------------|
| 모든 Pod → 모든 Pod 통신 허용 | Policy에 명시된 트래픽만 허용 |

> 💡 Network Policy가 하나라도 적용되면, 해당 Pod은 **명시적으로 허용된 트래픽만** 받을 수 있어요 (Default Deny).

### Ingress vs Egress

| 방향 | 설명 | 예시 |
|------|------|------|
| **Ingress** | Pod으로 **들어오는** 트래픽 | 외부 → 웹서버 Pod |
| **Egress** | Pod에서 **나가는** 트래픽 | 앱 Pod → DB Pod |

### Network Policy 예시: DB Pod 보호

DB Pod에 접근할 수 있는 트래픽을 API Pod의 3306 포트로만 제한하는 예시예요.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db              # 이 Policy가 적용될 Pod (DB)
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod     # API Pod에서 오는 트래픽만 허용
    ports:
    - protocol: TCP
      port: 3306
```

### 다양한 트래픽 소스 제어

```yaml
ingress:
- from:
  # 1. 특정 Pod에서 오는 트래픽
  - podSelector:
      matchLabels:
        name: api-pod

  # 2. 특정 네임스페이스에서 오는 트래픽
  - namespaceSelector:
      matchLabels:
        name: production

  # 3. 특정 IP 대역에서 오는 트래픽
  - ipBlock:
      cidr: 192.168.1.0/24
      except:
      - 192.168.1.100/32
```

> 💡 **주의**: `from` 배열 안에서 `-`으로 분리된 항목은 **OR** 조건이에요. 하나의 항목 안에 `podSelector`와 `namespaceSelector`를 같이 넣으면 **AND** 조건이에요.

```yaml
# OR: api-pod 이거나 production 네임스페이스
- from:
  - podSelector:
      matchLabels:
        name: api-pod
  - namespaceSelector:
      matchLabels:
        name: production

# AND: production 네임스페이스의 api-pod만
- from:
  - podSelector:
      matchLabels:
        name: api-pod
    namespaceSelector:
      matchLabels:
        name: production
```

### Egress 정책

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress-policy
spec:
  podSelector:
    matchLabels:
      name: api-pod
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: db
    ports:
    - protocol: TCP
      port: 3306
  - to:                        # DNS 허용 (필수!)
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

> 💡 Egress 정책을 설정할 때 **DNS (포트 53)**를 허용하는 걸 잊지 마세요! DNS를 차단하면 서비스 이름으로 통신이 안 돼요.

### Network Policy 지원 CNI

모든 CNI 플러그인이 Network Policy를 지원하지는 않아요.

| CNI | Network Policy 지원 |
|-----|-------------------|
| **Calico** | O |
| **Cilium** | O |
| **Weave Net** | O |
| **Flannel** | **X** |

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **ClusterRole** | 클러스터 전체 범위, nodes/PV 등 비 네임스페이스 리소스 관리 |
| **Image Security** | `imagePullSecrets`로 프라이빗 레지스트리 인증 |
| **SecurityContext** | runAsUser, capabilities, readOnlyRootFilesystem 등 보안 설정 |
| **Capabilities** | 컨테이너 레벨에서만 설정 가능, `add`/`drop` |
| **NetworkPolicy** | Ingress/Egress 트래픽 제어, podSelector + namespaceSelector + ipBlock |
| **AND vs OR** | 같은 항목 = AND, 분리된 항목 = OR |

이것으로 K8s 보안 시리즈를 마무리할게요! Security Primitives부터 TLS, RBAC, NetworkPolicy까지 — CKA 보안 영역의 핵심을 모두 다뤘어요. 🦥
