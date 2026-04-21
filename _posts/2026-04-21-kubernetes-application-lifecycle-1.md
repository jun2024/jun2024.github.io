---
title: "[Kubernetes] K8s 애플리케이션 생명주기 관리 (1) — Rolling Update, Rollback, Commands & Arguments"
excerpt: "Deployment의 Rolling Update 전략부터 Rollback, 그리고 Docker/K8s에서 컨테이너 명령어를 제어하는 방법까지 — 애플리케이션 배포의 핵심을 정리해보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Deployment, RollingUpdate, CKA, DevOps]

permalink: /kubernetes/kubernetes-application-lifecycle-1/

toc: true
toc_sticky: true

date: 2026-04-21
last_modified_at: 2026-04-21
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Application Lifecycle Management 섹션을 기반으로 정리한 내용이에요.
> 다음 포스트에서는 ConfigMap, Secret 등 애플리케이션 설정 관리를 다뤄요!

---

## 🦥 Deployment의 배포 전략 (Rollout)

### Rollout이란?

Deployment를 생성하거나 컨테이너 이미지를 업데이트하면 **Rollout**이 트리거돼요. Rollout은 새로운 버전의 Pod을 점진적으로 배포하는 과정이에요.

```bash
# Rollout 상태 확인
kubectl rollout status deployment/my-app

# Rollout 히스토리 확인
kubectl rollout history deployment/my-app
```

```
deployment.apps/my-app
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=my-app.yaml
2         kubectl set image deployment/my-app nginx=nginx:1.25
```

> 💡 **CKA 포인트**: `CHANGE-CAUSE`에 기록을 남기려면 배포 시 `--record` 플래그를 사용하거나, `kubernetes.io/change-cause` annotation을 추가하면 돼요. (`--record`는 deprecated되었지만 시험에서는 아직 나올 수 있어요.)

### 배포 전략 2가지

Deployment는 두 가지 배포 전략을 지원해요.

| 전략 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **Recreate** | 기존 Pod을 모두 삭제 후 새 Pod 생성 | 구현 간단, 리소스 충돌 없음 | **다운타임 발생** |
| **RollingUpdate** (기본값) | 기존 Pod을 하나씩 교체 | **무중단 배포** | 일시적으로 두 버전 공존 |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate         # 또는 Recreate
    rollingUpdate:
      maxSurge: 1               # 최대 추가 Pod 수 (기본: 25%)
      maxUnavailable: 1         # 최대 비가용 Pod 수 (기본: 25%)
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
```

### RollingUpdate 동작 과정

`maxSurge: 1`, `maxUnavailable: 1`, `replicas: 3`인 경우의 업데이트 과정이에요.

| 단계 | Old Pod | New Pod | 총 Pod | 설명 |
|------|---------|---------|--------|------|
| 시작 | 3 | 0 | 3 | 기존 버전 운영 중 |
| 1 | 2 | 1 | 3 | Old 1개 종료 + New 1개 생성 |
| 2 | 1 | 2 | 3 | Old 1개 종료 + New 1개 생성 |
| 3 | 0 | 3 | 3 | 모든 Pod이 새 버전으로 교체 완료 |

> 💡 RollingUpdate 내부에서는 **새로운 ReplicaSet**을 생성하고, 기존 ReplicaSet의 replicas를 줄이면서 새 ReplicaSet의 replicas를 늘리는 방식으로 동작해요.

---

## 🦥 Deployment 업데이트 방법

### 방법 1: YAML 파일 수정 후 apply

```yaml
# deployment.yaml의 이미지 버전을 수정
spec:
  containers:
  - name: nginx
    image: nginx:1.25    # 1.24 → 1.25로 변경
```

```bash
kubectl apply -f deployment.yaml
```

### 방법 2: kubectl set image

```bash
kubectl set image deployment/my-app nginx=nginx:1.25
```

### 방법 3: kubectl edit

```bash
kubectl edit deployment my-app
# 에디터에서 이미지 버전을 직접 수정 후 저장
```

> 💡 **CKA 팁**: 시험에서 빠르게 이미지를 변경할 때는 `kubectl set image` 방법이 가장 빨라요!

---

## 🦥 Rollback (롤백)

배포한 새 버전에 문제가 있으면 이전 버전으로 되돌릴 수 있어요.

### 마지막 버전으로 롤백

```bash
kubectl rollout undo deployment/my-app
```

### 특정 리비전으로 롤백

```bash
# 히스토리 확인
kubectl rollout history deployment/my-app

# 특정 리비전의 상세 정보 확인
kubectl rollout history deployment/my-app --revision=2

# 특정 리비전으로 롤백
kubectl rollout undo deployment/my-app --to-revision=1
```

### Rollout 일시 중지 및 재개

대규모 변경을 할 때 여러 수정사항을 한 번에 반영하고 싶으면, Rollout을 일시 중지할 수 있어요.

```bash
# 롤아웃 일시 중지
kubectl rollout pause deployment/my-app

# 여러 변경사항 적용
kubectl set image deployment/my-app nginx=nginx:1.25
kubectl set resources deployment/my-app -c=nginx --limits=cpu=200m,memory=512Mi

# 롤아웃 재개 (한 번에 반영)
kubectl rollout resume deployment/my-app
```

### 롤백 관련 CKA 빈출 명령어 정리

| 명령어 | 설명 |
|--------|------|
| `kubectl rollout status deployment/my-app` | Rollout 진행 상태 확인 |
| `kubectl rollout history deployment/my-app` | Rollout 히스토리 조회 |
| `kubectl rollout undo deployment/my-app` | 직전 버전으로 롤백 |
| `kubectl rollout undo deployment/my-app --to-revision=N` | 특정 리비전으로 롤백 |
| `kubectl rollout pause deployment/my-app` | 롤아웃 일시 중지 |
| `kubectl rollout resume deployment/my-app` | 롤아웃 재개 |

---

## 🦥 Docker에서의 Commands와 Arguments

K8s에서 컨테이너 명령어를 이해하려면 먼저 Docker의 `ENTRYPOINT`와 `CMD`를 알아야 해요.

### Dockerfile의 CMD와 ENTRYPOINT

```dockerfile
# CMD만 사용 — 컨테이너 시작 시 실행할 기본 명령어
FROM ubuntu
CMD ["sleep", "5"]
```

```dockerfile
# ENTRYPOINT 사용 — 항상 실행되는 명령어 (인자만 변경 가능)
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]            # ENTRYPOINT의 기본 인자
```

### CMD vs ENTRYPOINT 차이

| 항목 | CMD | ENTRYPOINT |
|------|-----|-----------|
| **역할** | 기본 실행 명령어 또는 ENTRYPOINT의 기본 인자 | 컨테이너 시작 시 항상 실행되는 명령어 |
| **docker run 인자** | CMD 전체가 **덮어씌워짐** | 인자가 ENTRYPOINT 뒤에 **추가됨** |
| **덮어쓰기** | `docker run <image> <cmd>` | `docker run --entrypoint <cmd> <image>` |

**예시로 이해해볼게요.**

```dockerfile
# Dockerfile
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

| docker run 명령어 | 실제 실행되는 명령어 |
|-------------------|-------------------|
| `docker run my-image` | `sleep 5` (ENTRYPOINT + CMD 기본값) |
| `docker run my-image 10` | `sleep 10` (CMD가 10으로 대체) |
| `docker run --entrypoint echo my-image hello` | `echo hello` (ENTRYPOINT 자체가 대체) |

> 💡 핵심 정리: **ENTRYPOINT**는 "무엇을 실행할지", **CMD**는 "기본 인자가 무엇인지"를 정의해요.

---

## 🦥 Kubernetes에서의 Command와 Args

Docker의 ENTRYPOINT/CMD가 K8s에서는 `command`와 `args`로 매핑돼요.

### Docker → K8s 매핑

| Docker | Kubernetes | 설명 |
|--------|-----------|------|
| `ENTRYPOINT` | `command` | 컨테이너 시작 시 실행할 명령어 |
| `CMD` | `args` | 명령어에 전달할 인자 |

### 기본 사용 예시

```yaml
# Docker ENTRYPOINT ["sleep"], CMD ["5"]를 K8s에서 오버라이드
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep"]        # ENTRYPOINT를 오버라이드
    args: ["10"]              # CMD를 오버라이드 (5 → 10)
```

### args만 오버라이드 (CMD만 변경)

Dockerfile에 `ENTRYPOINT ["sleep"]`과 `CMD ["5"]`가 있을 때, 인자만 바꾸고 싶으면 `args`만 지정하면 돼요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]              # CMD만 오버라이드 (sleep 10이 실행됨)
```

### command만 오버라이드 (ENTRYPOINT 변경)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]     # ENTRYPOINT를 완전히 대체
    args: ["10"]
```

### 다양한 command/args 작성 형식

```yaml
# 형식 1: 배열
command: ["sleep", "5000"]

# 형식 2: 각각 분리
command: ["sleep"]
args: ["5000"]

# 형식 3: 여러 줄
command:
- "sleep"
- "5000"

# 형식 4: 셸 명령어 실행
command: ["/bin/sh", "-c", "echo Hello && sleep 5000"]
```

### CKA 시험에서 자주 나오는 패턴

```yaml
# 패턴 1: 단순 sleep Pod
apiVersion: v1
kind: Pod
metadata:
  name: sleep-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]

# 패턴 2: 환경변수를 활용한 명령어
apiVersion: v1
kind: Pod
metadata:
  name: env-cmd-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c"]
    args: ["echo $(MY_VAR) && sleep 3600"]
    env:
    - name: MY_VAR
      value: "Hello CKA"
```

> 💡 **CKA 팁**: `command`를 지정하면 Dockerfile의 ENTRYPOINT가 **완전히 무시**돼요. `args`만 지정하면 CMD만 대체하고 ENTRYPOINT는 유지돼요. 이 차이를 정확히 알아야 시험에서 실수하지 않아요!

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Rollout** | Deployment 업데이트 시 자동 트리거, `kubectl rollout status`로 확인 |
| **배포 전략** | Recreate (다운타임 O) vs RollingUpdate (무중단, 기본값) |
| **maxSurge/maxUnavailable** | RollingUpdate 속도 제어, 기본값 25% |
| **Rollback** | `kubectl rollout undo`로 이전 버전 복구, `--to-revision`으로 특정 버전 |
| **Docker CMD/ENTRYPOINT** | CMD는 기본 인자, ENTRYPOINT는 실행 명령어 |
| **K8s command/args** | command = ENTRYPOINT, args = CMD 오버라이드 |

다음 포스트에서는 **환경변수, ConfigMap, Secret**을 활용한 애플리케이션 설정 관리를 다뤄볼게요! 🦥
