---
title: "[Kubernetes] K8s Self Healing — 자가 치유 애플리케이션의 모든 것"
excerpt: "Pod이 죽어도 자동으로 살아나는 K8s의 Self Healing 메커니즘 — ReplicaSet, Deployment, Liveness/Readiness/Startup Probe까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, SelfHealing, Probe, CKA, DevOps]

permalink: /kubernetes/kubernetes-self-healing/

toc: true
toc_sticky: true

date: 2026-04-28
last_modified_at: 2026-04-28
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Application Lifecycle Management 섹션을 기반으로 정리한 내용이에요.

---

## 🦥 Self Healing이란?

K8s의 가장 강력한 기능 중 하나는 **Self Healing (자가 치유)**이에요. 컨테이너가 실패하거나, 노드가 다운되거나, 헬스 체크에 실패해도 K8s가 자동으로 복구해줘요.

K8s는 **선언적 상태(Desired State)**와 **현재 상태(Current State)**를 지속적으로 비교하면서, 차이가 생기면 자동으로 현재 상태를 선언적 상태에 맞춰요. 이것이 Self Healing의 핵심 원리예요.

| 상황 | K8s의 자동 대응 |
|------|---------------|
| 컨테이너 크래시 | 자동 재시작 (restartPolicy) |
| Pod 삭제됨 | ReplicaSet이 새 Pod 생성 |
| 노드 다운 | 다른 노드에 Pod 재스케줄링 |
| 헬스 체크 실패 | 컨테이너 재시작 (Liveness Probe) |
| 트래픽 수신 불가 | Service에서 제외 (Readiness Probe) |

---

## 🦥 ReplicaSet & Deployment의 Self Healing

### ReplicaSet의 자동 복구

ReplicaSet은 항상 **지정된 수의 Pod**이 실행되도록 보장해요.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:1.25
```

```bash
# 3개 Pod 실행 중
kubectl get pods
# my-app-rs-abc12   1/1   Running   0   5m
# my-app-rs-def34   1/1   Running   0   5m
# my-app-rs-ghi56   1/1   Running   0   5m

# Pod 하나를 수동 삭제
kubectl delete pod my-app-rs-abc12

# 즉시 새 Pod이 자동 생성됨
kubectl get pods
# my-app-rs-def34   1/1   Running   0   5m
# my-app-rs-ghi56   1/1   Running   0   5m
# my-app-rs-jkl78   1/1   Running   0   3s    ← 자동 생성!
```

### Deployment의 Self Healing

Deployment는 ReplicaSet 위에 Rolling Update, Rollback 기능을 추가한 리소스예요. 실무에서는 ReplicaSet을 직접 사용하지 않고 Deployment를 사용해요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:1.25
```

Deployment가 보장하는 Self Healing 동작은 이래요.

| 이벤트 | 자동 대응 |
|--------|----------|
| Pod 크래시 | 컨테이너 자동 재시작 |
| Pod 삭제 | 새 Pod 자동 생성 (replicas 유지) |
| 노드 장애 | 다른 노드에 Pod 재생성 (5분 대기 후) |
| 이미지 업데이트 실패 | Rollback 가능 상태 유지 |

---

## 🦥 restartPolicy — 컨테이너 재시작 정책

Pod 레벨에서 컨테이너의 재시작 정책을 설정할 수 있어요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  restartPolicy: Always    # Always | OnFailure | Never
  containers:
  - name: my-app
    image: my-app:1.0
```

| 정책 | 설명 | 사용 시기 |
|------|------|----------|
| **Always** (기본값) | 컨테이너가 종료되면 항상 재시작 | 일반적인 서비스 (웹서버, API 등) |
| **OnFailure** | 비정상 종료(exit code ≠ 0)일 때만 재시작 | 배치 작업, Job |
| **Never** | 재시작하지 않음 | 일회성 작업, 디버깅 |

> 💡 **CKA 포인트**: Deployment/ReplicaSet의 Pod 템플릿에서는 `restartPolicy`가 반드시 `Always`여야 해요. `OnFailure`나 `Never`를 사용하면 생성 에러가 나요. Job/CronJob에서는 `OnFailure`나 `Never`를 사용해요.

### 재시작 백오프 (CrashLoopBackOff)

컨테이너가 반복적으로 실패하면 K8s는 재시작 간격을 점점 늘려요.

| 재시작 횟수 | 대기 시간 |
|-----------|----------|
| 1회 | 즉시 |
| 2회 | 10초 |
| 3회 | 20초 |
| 4회 | 40초 |
| N회 | 최대 5분까지 지수 증가 |

이 상태가 **CrashLoopBackOff**예요. `kubectl describe pod`로 원인을 확인하고, `kubectl logs --previous`로 이전 컨테이너 로그를 확인해야 해요.

---

## 🦥 Liveness Probe — "컨테이너가 살아있는가?"

Liveness Probe는 컨테이너가 정상 동작하는지를 주기적으로 확인해요. 실패하면 K8s가 컨테이너를 **재시작**해요.

### 왜 필요한가?

프로세스가 살아있지만 **실제로는 동작하지 않는 경우**가 있어요. 예를 들어 애플리케이션이 데드락에 빠지거나, 무한 루프에 걸린 경우 프로세스는 죽지 않지만 요청을 처리할 수 없어요. Liveness Probe가 이런 상황을 감지해서 컨테이너를 재시작해줘요.

### Probe 방식 3가지

**1. HTTP GET**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15     # 컨테이너 시작 후 첫 검사까지 대기
      periodSeconds: 10           # 검사 주기
      timeoutSeconds: 3           # 응답 대기 시간
      failureThreshold: 3        # 연속 실패 횟수 (이후 재시작)
      successThreshold: 1        # 성공으로 판단할 횟수
```

**2. TCP Socket**

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 10
```

**3. Exec Command**

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Probe 파라미터 정리

| 파라미터 | 기본값 | 설명 |
|---------|--------|------|
| `initialDelaySeconds` | 0 | 컨테이너 시작 후 첫 Probe까지 대기 시간 |
| `periodSeconds` | 10 | Probe 실행 주기 |
| `timeoutSeconds` | 1 | Probe 응답 대기 시간 |
| `failureThreshold` | 3 | 연속 실패 시 비정상으로 판단 |
| `successThreshold` | 1 | 연속 성공 시 정상으로 판단 |

---

## 🦥 Readiness Probe — "트래픽을 받을 준비가 됐는가?"

Readiness Probe는 컨테이너가 트래픽을 처리할 준비가 됐는지 확인해요. 실패하면 **Service의 Endpoints에서 해당 Pod을 제거**해서 트래픽이 가지 않게 해요. 컨테이너를 재시작하지는 않아요.

### Liveness vs Readiness 비교

| 항목 | Liveness Probe | Readiness Probe |
|------|---------------|----------------|
| **질문** | "살아있니?" | "트래픽 받을 준비 됐니?" |
| **실패 시 동작** | 컨테이너 **재시작** | Service Endpoints에서 **제거** |
| **용도** | 데드락, 무한루프 감지 | 초기화 중, 일시적 과부하 |
| **재시작 여부** | O | X |

### 사용 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-probes
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
    # Readiness: 준비 됐는지 확인
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    # Liveness: 살아있는지 확인
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
```

> 💡 **실무 팁**: Liveness와 Readiness Probe를 함께 사용하되, Liveness의 `initialDelaySeconds`를 Readiness보다 길게 설정하세요. 앱이 아직 초기화 중인데 Liveness가 먼저 실패하면 불필요한 재시작이 발생해요.

---

## 🦥 Startup Probe — "앱이 시작됐는가?"

Startup Probe는 **시작이 느린 애플리케이션**을 위한 Probe예요. Startup Probe가 성공할 때까지 Liveness/Readiness Probe가 실행되지 않아요.

### 왜 필요한가?

레거시 앱이나 무거운 앱은 시작하는 데 30초~수분이 걸릴 수 있어요. Liveness Probe의 `initialDelaySeconds`를 너무 길게 설정하면, 실제 장애 감지도 그만큼 늦어져요. Startup Probe를 사용하면 이 문제를 해결할 수 있어요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-start-app
spec:
  containers:
  - name: app
    image: legacy-app:1.0
    ports:
    - containerPort: 8080
    # Startup: 앱 시작 완료 확인 (최대 300초 대기)
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30       # 30 × 10초 = 최대 300초
      periodSeconds: 10
    # Liveness: Startup 성공 후에만 동작
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
    # Readiness: Startup 성공 후에만 동작
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

### 3가지 Probe 실행 순서

| 순서 | Probe | 동작 |
|------|-------|------|
| 1 | **Startup Probe** | 앱 시작 완료 확인 (성공할 때까지 다른 Probe 비활성) |
| 2 | **Readiness Probe** | 트래픽 수신 가능 여부 확인 → Service Endpoints 등록/제거 |
| 3 | **Liveness Probe** | 컨테이너 정상 동작 확인 → 실패 시 재시작 |

---

## 🦥 노드 장애 시 Self Healing

### Pod Eviction Timeout

노드가 다운되면 K8s는 즉시 Pod을 재생성하지 않아요. **기본 5분(pod-eviction-timeout)**을 기다린 후 해당 노드의 Pod을 다른 노드로 재스케줄링해요.

| 단계 | 시간 | 동작 |
|------|------|------|
| 노드 다운 | 0초 | kubelet이 heartbeat 전송 중단 |
| NotReady 전환 | ~40초 | Node Condition이 Ready → NotReady |
| Pod Eviction | ~5분 | kube-controller-manager가 Pod을 Evict |
| 재스케줄링 | 5분+ | 다른 노드에 새 Pod 생성 |

> 💡 Deployment/ReplicaSet으로 관리되는 Pod만 다른 노드에 재생성돼요. 단독 Pod(`kind: Pod`으로 직접 생성한 것)은 재생성되지 않아요!

### PodDisruptionBudget (PDB)

노드 유지보수 시 Pod이 한꺼번에 사라지는 것을 방지하는 리소스예요.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2          # 최소 2개 Pod은 항상 유지
  # 또는 maxUnavailable: 1  # 최대 1개까지만 비가용 허용
  selector:
    matchLabels:
      app: my-app
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Self Healing 원리** | Desired State vs Current State를 지속 비교, 자동 복구 |
| **ReplicaSet/Deployment** | 지정된 replicas 수를 항상 유지, Pod 삭제 시 자동 재생성 |
| **restartPolicy** | Always (기본), OnFailure (Job), Never (일회성) |
| **Liveness Probe** | "살아있니?" → 실패 시 컨테이너 재시작 |
| **Readiness Probe** | "준비 됐니?" → 실패 시 Service에서 제외 |
| **Startup Probe** | "시작 완료?" → 성공 전까지 Liveness/Readiness 비활성 |
| **노드 장애** | 5분 대기 후 다른 노드에 재스케줄링 (Deployment만) |

다음 포스트에서는 **클러스터 유지보수 — OS Upgrades와 Cluster Upgrade**를 다뤄볼게요! 🦥
