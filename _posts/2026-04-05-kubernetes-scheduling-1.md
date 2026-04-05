---
title: "[Kubernetes] K8s 스케줄링 완벽 정리 (1) — Pod 배치 전략의 모든 것"
excerpt: "Manual Scheduling부터 Labels, Taints/Tolerations, Node Affinity까지 — Pod이 어떤 노드에 배치되는지 제어하는 방법을 총정리해보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Scheduling, CKA, DevOps]

permalink: /kubernetes/kubernetes-scheduling-1/

toc: true
toc_sticky: true

date: 2026-04-05
last_modified_at: 2026-04-05
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Scheduling 섹션을 기반으로 정리한 내용이에요.
> 다음 포스트에서는 Resource Limits, DaemonSet, Static Pod, Multiple Scheduler를 다뤄요!

---

## 🦥 스케줄러는 어떻게 동작할까?

Pod을 생성하면 K8s 스케줄러가 자동으로 "이 Pod을 어떤 노드에 올릴까?" 를 결정해요. 그 과정은 이래요.

```
Pod 생성 요청
  → kube-scheduler가 감지
    → 필터링 (Filtering): 조건에 안 맞는 노드 제외
      → 스코어링 (Scoring): 남은 노드 중 최적 노드 선정
        → 바인딩 (Binding): Pod의 nodeName 필드에 노드 할당
```

모든 Pod에는 `nodeName`이라는 필드가 있는데, 기본값은 비어 있어요. 스케줄러가 노드를 결정하면 이 필드에 노드 이름을 채워 넣는 방식이에요. 이 과정을 이해하면, 아래에서 다룰 다양한 스케줄링 기법들이 왜 필요한지 자연스럽게 알 수 있어요!

---

## 🦥 Manual Scheduling (수동 스케줄링)

스케줄러가 없거나, 특정 노드에 Pod을 직접 지정하고 싶을 때 사용해요.

### 방법 1: nodeName 직접 지정

Pod 정의 파일에 `nodeName`을 명시하면 스케줄러를 거치지 않고 해당 노드에 바로 배치돼요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 8080
  nodeName: node02  # 이 Pod은 무조건 node02에 배치
```

> **주의!** `nodeName`은 Pod 생성 시에만 지정할 수 있어요. 이미 생성된 Pod의 `nodeName`은 변경할 수 없어요.

### 방법 2: Binding 오브젝트

이미 생성되어 Pending 상태인 Pod을 특정 노드에 배치하고 싶을 때 사용해요. API 서버에 Binding 오브젝트를 POST로 보내면 돼요.

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node02
```

```bash
# Binding 오브젝트를 JSON으로 변환해서 API 서버에 전송
curl --header "Content-Type:application/json" \
  --request POST \
  --data '{"apiVersion":"v1","kind":"Binding","metadata":{"name":"nginx"},"target":{"apiVersion":"v1","kind":"Node","name":"node02"}}' \
  http://localhost:8001/api/v1/namespaces/default/pods/nginx/binding/
```

실무에서 Manual Scheduling을 직접 쓸 일은 많지 않지만, 스케줄러 장애 시 응급 조치나 디버깅할 때 알아두면 유용해요.

---

## 🦥 Labels & Selectors (라벨과 셀렉터)

**Labels**은 K8s 오브젝트에 붙이는 키-값 쌍 메타데이터예요. **Selectors**는 이 라벨을 기준으로 오브젝트를 필터링하는 메커니즘이고요. 스케줄링에서도 핵심적인 역할을 해요.

### 라벨 정의하기

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
    tier: presentation
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 8080
```

### 라벨로 필터링하기

```bash
# 특정 라벨을 가진 Pod 조회
kubectl get pods --selector app=App1

# 여러 라벨 조건 (AND)
kubectl get pods --selector app=App1,tier=presentation

# 모든 오브젝트에서 라벨 기준 필터링
kubectl get all --selector env=prod
```

### ReplicaSet에서 라벨 활용

ReplicaSet은 `selector.matchLabels`로 관리할 Pod을 식별해요. 여기서 라벨이 매칭 기준이 돼요.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1         # ReplicaSet 자체의 라벨
    function: Front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1       # 이 라벨을 가진 Pod을 관리
  template:
    metadata:
      labels:
        app: App1     # Pod에 붙는 라벨 (selector와 일치해야 함)
        function: Front-end
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp
```

### Service에서 라벨 활용

Service도 `selector`로 트래픽을 보낼 Pod을 선택해요.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: App1         # app=App1 라벨을 가진 Pod에 트래픽 전달
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

### Annotations (어노테이션)

라벨과 비슷하게 생겼지만 목적이 달라요. 라벨은 **선택/필터링** 용도이고, 어노테이션은 **정보 기록** 용도예요. 셀렉터로 사용할 수 없어요.

```yaml
metadata:
  name: simple-webapp
  labels:
    app: App1
  annotations:
    buildversion: "1.34"      # 빌드 버전 기록
    contact: "team-backend"   # 담당 팀 기록
```

---

## 🦥 Taints & Tolerations (테인트와 톨러레이션)

**Taint**는 노드에 거는 "출입 제한"이고, **Toleration**은 Pod에 붙이는 "출입 허가증"이에요.

핵심 포인트는 이거예요: Taints/Tolerations는 **Pod을 특정 노드로 끌어당기는 게 아니라, 조건에 안 맞는 Pod을 밀어내는 것**이에요!

### Taint 적용하기

```bash
# 문법
kubectl taint nodes <node-name> <key>=<value>:<effect>

# 예시: node1에 app=blue:NoSchedule 테인트 적용
kubectl taint nodes node1 app=blue:NoSchedule
```

### 3가지 Taint Effect

| Effect | 동작 |
|---|---|
| `NoSchedule` | 톨러레이션 없는 Pod은 **절대** 스케줄링 안 됨 |
| `PreferNoSchedule` | 가급적 스케줄링 안 하지만 **다른 노드가 없으면 허용** |
| `NoExecute` | 새 Pod 스케줄링 차단 + **이미 실행 중인 Pod도 퇴출** |

`NoExecute`가 가장 강력해요. 이미 돌고 있는 Pod이라도 톨러레이션이 없으면 쫓아내요!

### Toleration 정의하기

Pod이 테인트가 걸린 노드에 배치되려면 매칭되는 톨러레이션을 가지고 있어야 해요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

> **주의!** 톨러레이션 값들은 반드시 **큰따옴표("")** 로 감싸야 해요.

### Taint 확인/제거

```bash
# 노드의 테인트 확인
kubectl describe node node1 | grep -i taint

# 테인트 제거 (끝에 - 붙이기)
kubectl taint nodes node1 app=blue:NoSchedule-
```

### Master 노드의 Taint

K8s는 기본적으로 Master 노드에 `NoSchedule` 테인트를 걸어둬요. 그래서 일반 Pod은 Master에 스케줄링되지 않아요.

```bash
# Master 노드 테인트 확인
kubectl describe node kubemaster | grep Taint
# 출력: node-role.kubernetes.io/master:NoSchedule
```

---

## 🦥 Node Selectors (노드 셀렉터)

특정 라벨을 가진 노드에만 Pod을 배치하고 싶을 때 사용하는 가장 **간단한** 방법이에요.

### 사용법

**1단계: 노드에 라벨 붙이기**

```bash
kubectl label nodes node-1 size=Large
```

**2단계: Pod에 nodeSelector 지정**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  nodeSelector:
    size: Large   # size=Large 라벨을 가진 노드에만 배치
```

### 한계

nodeSelector는 단순 등호(=) 매칭만 가능해요. 이런 복잡한 조건은 표현할 수 없어요:

- "Large **또는** Medium 노드에 배치" → 불가능
- "Small이 **아닌** 노드에 배치" → 불가능

이런 복잡한 조건이 필요하면 **Node Affinity**를 써야 해요!

---

## 🦥 Node Affinity (노드 어피니티)

Node Affinity는 nodeSelector의 **상위 호환**이에요. 훨씬 유연한 조건으로 Pod의 노드 배치를 제어할 수 있어요.

### 기본 예시 — In 연산자

"Large **또는** Medium 노드에 배치해줘"

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
```

### NotIn 연산자

"Small이 **아닌** 노드에 배치해줘"

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: size
          operator: NotIn
          values:
          - Small
```

### Exists 연산자

"size 라벨이 **존재하는** 노드에 배치해줘" (값은 상관없음)

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: size
          operator: Exists
```

### Affinity 타입 (이름이 길지만 중요해요!)

| 타입 | DuringScheduling | DuringExecution | 설명 |
|---|---|---|---|
| `requiredDuringScheduling IgnoredDuringExecution` | **Required** | Ignored | 스케줄링 시 반드시 조건 충족해야 함. 실행 중에는 무시 |
| `preferredDuringScheduling IgnoredDuringExecution` | **Preferred** | Ignored | 조건 충족 노드를 선호하지만, 없으면 다른 노드에도 배치 |

- **DuringScheduling**: Pod이 처음 배치될 때 규칙을 적용할지
  - `Required`: 조건에 맞는 노드가 없으면 Pod은 Pending 상태 유지
  - `Preferred`: 가급적 맞추되, 안 되면 아무 노드에나 배치
- **DuringExecution**: 이미 실행 중인 Pod에 대해 규칙을 적용할지
  - `Ignored`: 이미 돌고 있는 Pod은 건드리지 않음

---

## 🦥 Taints/Tolerations vs Node Affinity

둘 다 Pod 배치를 제어하지만, **해결하는 문제가 달라요**.

| 비교 | Taints/Tolerations | Node Affinity |
|---|---|---|
| **방향** | 노드가 Pod을 **밀어냄** (배척) | Pod이 노드를 **끌어당김** (선호) |
| **보장하는 것** | 원치 않는 Pod이 오는 걸 막음 | 원하는 노드에 Pod을 배치 |
| **보장 못 하는 것** | Pod이 반드시 그 노드에 감 | 다른 Pod이 그 노드에 오는 걸 막음 |

### 둘 다 쓰면?

**Taint + Node Affinity를 조합**하면 **완벽한 노드 전용화(Node Dedication)**가 가능해요!

```
시나리오: Blue 팀 전용 노드를 만들고 싶다

1. 노드에 Taint 적용 → Blue 톨러레이션 없는 Pod은 진입 불가
   kubectl taint nodes node1 team=blue:NoSchedule

2. 노드에 Label 적용
   kubectl label nodes node1 team=blue

3. Blue 팀 Pod에 Toleration + Node Affinity 둘 다 적용
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: blue-app
spec:
  containers:
  - name: blue-app
    image: blue-app
  # Toleration: blue 테인트가 걸린 노드에 배치 허용
  tolerations:
  - key: "team"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
  # Node Affinity: team=blue 라벨의 노드에만 배치
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: team
            operator: In
            values:
            - blue
```

이렇게 하면:

- **Taint**가 다른 팀의 Pod이 Blue 노드에 오는 걸 막고
- **Node Affinity**가 Blue Pod이 다른 노드로 가는 걸 막아요

결과적으로 Blue 노드에는 Blue Pod만, Blue Pod은 Blue 노드에만 배치돼요!

---

## 🦥 마무리

정리하면 Pod 배치 전략은 이렇게 선택하면 돼요.

| 상황 | 사용할 기법 |
|---|---|
| 특정 노드에 긴급 배치 (스케줄러 없이) | Manual Scheduling (`nodeName`) |
| 오브젝트 분류/필터링 | Labels & Selectors |
| 특정 노드에 원치 않는 Pod 차단 | Taints & Tolerations |
| 특정 노드에 Pod 유인 (단순) | Node Selectors |
| 특정 노드에 Pod 유인 (복잡 조건) | Node Affinity |
| 노드를 특정 워크로드 전용으로 | Taint + Node Affinity 조합 |

다음 포스트에서는 [Resource Limits, DaemonSet, Static Pod, Multiple Scheduler](/kubernetes/kubernetes-scheduling-2/)를 다뤄볼게요!
