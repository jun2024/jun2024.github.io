---
title: "[Kubernetes] K8s 스케줄링 완벽 정리 (2) — 리소스 관리부터 커스텀 스케줄러까지"
excerpt: "Resource Limits, DaemonSet, Static Pod, Multiple Scheduler까지 — 스케줄링의 나머지 절반을 실전 예시와 함께 정리해보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Scheduling, CKA, DevOps]

permalink: /kubernetes/kubernetes-scheduling-2/

toc: true
toc_sticky: true

date: 2026-04-06
last_modified_at: 2026-04-06
---

> 이 포스트는 [K8s 스케줄링 완벽 정리 (1)](/kubernetes/kubernetes-scheduling-1/)에 이어지는 내용이에요.
> 이전 포스트에서 Manual Scheduling, Labels, Taints/Tolerations, Node Affinity를 다뤘으니 먼저 읽고 오는 걸 추천해요.

---

## 🦥 Resource Requests & Limits (리소스 요청과 제한)

스케줄러는 Pod을 노드에 배치할 때 **리소스(CPU, Memory)가 충분한 노드**를 골라요. 이때 기준이 되는 게 바로 `requests`와 `limits`예요.

### 핵심 개념

| 구분 | 의미 | 역할 |
|---|---|---|
| **Requests** | 컨테이너에 **보장되는** 최소 리소스 | 스케줄러가 노드 선택 시 참고 |
| **Limits** | 컨테이너가 사용할 수 있는 **최대** 리소스 | 초과 시 throttle(CPU) 또는 OOMKill(Memory) |

### CPU 단위 이해하기

| 표현 | 의미 |
|---|---|
| `1` | 1 vCPU (AWS) / 1 Core (GCP) / 1 하이퍼스레드 |
| `0.5` 또는 `500m` | 0.5 vCPU |
| `100m` | 0.1 vCPU (최솟값: `1m`) |

### Memory 단위 이해하기

| 표현 | 의미 |
|---|---|
| `256Mi` | 256 메비바이트 (≈ 268MB) |
| `1Gi` | 1 기비바이트 (≈ 1.07GB) |
| `128M` | 128 메가바이트 (주의: `Mi` ≠ `M`) |

> `Mi` (메비바이트, 1024 기반)와 `M` (메가바이트, 1000 기반)은 달라요! K8s에서는 보통 `Mi`, `Gi`를 사용해요.

### 기본 설정 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"   # 최소 256Mi 메모리 보장
        cpu: "500m"       # 최소 0.5 CPU 보장
      limits:
        memory: "512Mi"   # 최대 512Mi까지 사용 가능
        cpu: "1"          # 최대 1 CPU까지 사용 가능
```

### 리소스 초과 시 동작

| 리소스 | 초과 시 동작 |
|---|---|
| **CPU** | 스로틀링 (throttle) — 속도만 느려지고 Pod은 살아있음 |
| **Memory** | OOM Kill — Pod이 종료됨 (재시작 정책에 따라 재시작) |

CPU는 **압축 가능한 리소스**(compressible)라서 초과하면 속도만 줄이지만, Memory는 **압축 불가능**(incompressible)이라서 초과하면 Pod을 죽여요.

### LimitRange — 네임스페이스 기본값 설정

Pod마다 일일이 리소스를 지정하기 귀찮을 때, 네임스페이스 레벨에서 기본값을 설정할 수 있어요.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-defaults
  namespace: production
spec:
  limits:
  - default:           # 기본 limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:    # 기본 requests
      cpu: "250m"
      memory: "128Mi"
    max:               # 최대 허용값
      cpu: "2"
      memory: "2Gi"
    min:               # 최소 허용값
      cpu: "100m"
      memory: "64Mi"
    type: Container
```

> LimitRange는 **새로 생성되는 Pod**에만 적용돼요. 기존 Pod에는 영향 없어요.

### ResourceQuota — 네임스페이스 총량 제한

네임스페이스 전체에서 사용할 수 있는 리소스 총량을 제한하고 싶을 때 사용해요.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"       # 네임스페이스 전체 CPU requests 합계 4 이하
    requests.memory: "4Gi"  # 네임스페이스 전체 Memory requests 합계 4Gi 이하
    limits.cpu: "10"        # 네임스페이스 전체 CPU limits 합계 10 이하
    limits.memory: "10Gi"   # 네임스페이스 전체 Memory limits 합계 10Gi 이하
    pods: "20"              # 최대 Pod 수
```

### 리소스 설정 Best Practice

| 시나리오 | Requests | Limits | 설명 |
|---|---|---|---|
| **권장 (일반)** | 설정 | 설정 | 안정적인 리소스 관리 |
| **권장 (CPU)** | 설정 | 미설정 | CPU는 throttle만 되니까 limit 없이도 괜찮음 |
| **위험** | 미설정 | 미설정 | 한 Pod이 노드 리소스를 독점할 수 있음 |
| **비추천** | 미설정 | 설정 | K8s가 자동으로 requests=limits로 설정함 |

---

## 🦥 DaemonSet (데몬셋)

**DaemonSet**은 클러스터의 **모든 노드(또는 특정 노드)에 Pod을 딱 1개씩** 실행하는 오브젝트예요. 새 노드가 추가되면 자동으로 Pod이 생기고, 노드가 제거되면 Pod도 같이 사라져요.

### ReplicaSet과 뭐가 다를까?

| 비교 | ReplicaSet | DaemonSet |
|---|---|---|
| Pod 수 | 전체 클러스터에서 `replicas` 수 만큼 | **노드당 정확히 1개** |
| 노드 추가 시 | 자동 증가 안 됨 | 자동으로 새 노드에 Pod 배치 |
| 용도 | 일반 애플리케이션 | 노드별 에이전트/데몬 |

### 대표적인 사용 사례

- **모니터링 에이전트**: Prometheus Node Exporter, Datadog Agent
- **로그 수집기**: Fluentd, Filebeat
- **네트워크 플러그인**: kube-proxy, Calico, Cilium
- **스토리지 데몬**: Ceph, GlusterFS

사실 `kube-proxy`도 DaemonSet으로 배포돼요!

### YAML 정의

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  labels:
    app: monitoring-agent
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

> ReplicaSet과 YAML 구조가 거의 동일해요. `kind`만 `DaemonSet`으로 바꾸고 `replicas` 필드가 없는 게 차이예요.

### 주요 명령어

```bash
# DaemonSet 생성
kubectl apply -f daemonset-definition.yaml

# DaemonSet 목록 조회
kubectl get daemonsets

# 상세 정보 확인
kubectl describe daemonset monitoring-agent

# 모든 네임스페이스의 DaemonSet 확인
kubectl get daemonsets --all-namespaces
```

### 특정 노드에만 DaemonSet 배포

nodeSelector나 Node Affinity를 사용하면 특정 노드에만 DaemonSet Pod을 배포할 수 있어요.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-monitor
spec:
  selector:
    matchLabels:
      app: gpu-monitor
  template:
    metadata:
      labels:
        app: gpu-monitor
    spec:
      nodeSelector:
        gpu: "true"   # gpu=true 라벨이 있는 노드에만 배포
      containers:
      - name: gpu-monitor
        image: gpu-monitor:latest
```

---

## 🦥 Static Pods (스태틱 파드)

**Static Pod**은 kubelet이 **API 서버 없이 직접 관리**하는 Pod이에요. 특정 디렉토리에 YAML 파일을 놓으면 kubelet이 자동으로 감지해서 Pod을 생성해요.

### 동작 원리

```
일반 Pod:
  kubectl → API Server → Scheduler → kubelet → Container

Static Pod:
  YAML 파일 → kubelet → Container (API Server, Scheduler 불필요!)
```

kubelet은 지정된 디렉토리를 주기적으로 감시해요. 파일이 추가되면 Pod을 만들고, 파일이 삭제되면 Pod을 제거하고, 파일이 수정되면 Pod을 재생성해요.

### 설정 방법

**방법 1: kubelet 실행 옵션**

```bash
# kubelet 서비스 파일에 매니페스트 경로 지정
ExecStart=/usr/local/bin/kubelet \
  --pod-manifest-path=/etc/kubernetes/manifests \
  ...
```

**방법 2: kubelet 설정 파일**

```yaml
# /var/lib/kubelet/config.yaml
staticPodPath: /etc/kubernetes/manifests
```

```bash
# kubelet 설정 파일 경로 확인
ps aux | grep kubelet | grep -- --config

# Static Pod 매니페스트 경로 확인
cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

### Static Pod 예시

```yaml
# /etc/kubernetes/manifests/nginx-static.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-static
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

이 파일을 `/etc/kubernetes/manifests/` 에 놓기만 하면 kubelet이 자동으로 Pod을 생성해요!

### Static Pod 식별하기

```bash
# Static Pod은 이름 뒤에 노드 이름이 붙어요
kubectl get pods --all-namespaces

# 예시 출력:
# kube-system   etcd-master                   ← Static Pod (master 노드)
# kube-system   kube-apiserver-master          ← Static Pod (master 노드)
# kube-system   kube-scheduler-master          ← Static Pod (master 노드)
```

### 핵심 사용 사례: Control Plane 부트스트랩

K8s 클러스터의 핵심 컴포넌트들이 바로 Static Pod이에요!

| Static Pod | 역할 |
|---|---|
| `kube-apiserver` | API 서버 |
| `etcd` | 분산 저장소 |
| `kube-controller-manager` | 컨트롤러 매니저 |
| `kube-scheduler` | 스케줄러 |

"닭이 먼저냐 달걀이 먼저냐" 문제를 해결해줘요. API 서버가 없는 상태에서 API 서버 Pod을 어떻게 만들까? → Static Pod으로!

### Static Pods vs DaemonSets

| 비교 | Static Pods | DaemonSets |
|---|---|---|
| **관리 주체** | kubelet 단독 | API Server + Scheduler |
| **관리 범위** | 해당 노드만 | 클러스터 전체 |
| **스케줄러 관여** | 없음 | 있음 |
| **kubectl 가시성** | 읽기 전용 (미러 Pod) | 완전한 CRUD |
| **주 용도** | Control Plane 컴포넌트 | 노드별 에이전트 |

---

## 🦥 Multiple Schedulers (멀티 스케줄러)

K8s는 기본 스케줄러(`default-scheduler`) 외에 **커스텀 스케줄러를 추가로 실행**할 수 있어요. 특정 워크로드에 대해 다른 스케줄링 로직을 적용하고 싶을 때 유용해요.

### 사용 시나리오

- GPU 워크로드는 GPU 최적화 스케줄러로 처리
- 배치 작업은 bin-packing 스케줄러로 처리
- ML 학습은 gang-scheduling 스케줄러로 처리

### 커스텀 스케줄러 배포 (Pod 방식)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: k8s.gcr.io/kube-scheduler:v1.29.0
    command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=false                          # HA 환경이 아니면 false
    - --scheduler-name=my-custom-scheduler          # 스케줄러 이름 지정
    - --lock-object-name=my-custom-scheduler        # 리더 선출용 Lock 이름
```

### Deployment 방식 (프로덕션 권장)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: my-custom-scheduler
  template:
    metadata:
      labels:
        component: my-custom-scheduler
    spec:
      serviceAccountName: my-custom-scheduler
      containers:
      - name: kube-scheduler
        image: k8s.gcr.io/kube-scheduler:v1.29.0
        command:
        - kube-scheduler
        - --kubeconfig=/etc/kubernetes/scheduler.conf
        - --leader-elect=false
        - --scheduler-name=my-custom-scheduler
```

### Pod에서 커스텀 스케줄러 사용하기

`schedulerName` 필드로 어떤 스케줄러를 사용할지 지정해요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training-job
spec:
  schedulerName: my-custom-scheduler  # 커스텀 스케줄러 사용
  containers:
  - name: training
    image: gpu-training:latest
    resources:
      limits:
        nvidia.com/gpu: 1
```

`schedulerName`을 지정하지 않으면 `default-scheduler`가 처리해요.

### 확인 및 디버깅

```bash
# 스케줄러 Pod 확인
kubectl get pods -n kube-system | grep scheduler

# 스케줄링 이벤트 확인 — 어떤 스케줄러가 Pod을 배치했는지 볼 수 있음
kubectl get events -o wide | grep Scheduled

# 출력 예시:
# default   Pod   Scheduled   my-custom-scheduler   Successfully assigned default/gpu-job to node-gpu-01

# 스케줄러 로그 확인
kubectl logs my-custom-scheduler -n kube-system
```

---

## 🦥 Scheduler Profiles (스케줄러 프로필)

K8s 1.18+부터는 하나의 스케줄러 바이너리에서 **여러 프로필**을 정의할 수 있어요. 별도의 스케줄러를 배포하지 않고도 다양한 스케줄링 전략을 구현할 수 있어서 운영이 훨씬 간편해요.

### 스케줄링 파이프라인

스케줄러는 내부적으로 이런 단계를 거쳐요.

```
1. Scheduling Queue (큐잉)
   → PrioritySort: 우선순위 기반 정렬

2. Filtering (필터링)
   → NodeResourcesFit: 리소스 충분한 노드만 남김
   → NodeName: nodeName 매칭
   → NodeUnschedulable: Unschedulable 노드 제외

3. Scoring (점수 매기기)
   → NodeResourcesFit: 리소스 밸런스 점수
   → ImageLocality: 이미지가 이미 있는 노드 선호

4. Binding (바인딩)
   → DefaultBinder: Pod을 노드에 바인딩
```

### 멀티 프로필 설정

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  # 기본 프로필
  - schedulerName: default-scheduler
    plugins:
      score:
        enabled:
        - name: NodeResourcesBalancedAllocation
        - name: ImageLocality

  # 고밀도 패킹 프로필 — 노드를 최대한 꽉 채움
  - schedulerName: high-density-scheduler
    plugins:
      score:
        enabled:
        - name: NodeResourcesFit
          weight: 100   # 리소스 사용량이 높은 노드 선호
        disabled:
        - name: NodeResourcesBalancedAllocation

  # GPU 워크로드 전용 프로필
  - schedulerName: gpu-scheduler
    plugins:
      filter:
        enabled:
        - name: NodeResourcesFit
      score:
        enabled:
        - name: NodeResourcesFit
```

```yaml
# 고밀도 패킹 스케줄러 사용
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  schedulerName: high-density-scheduler
  containers:
  - name: batch
    image: batch-processor
```

---

## 🦥 전체 스케줄링 기법 요약

지금까지 다룬 내용을 한눈에 정리해볼게요.

| 기법 | 목적 | 핵심 포인트 |
|---|---|---|
| **Manual Scheduling** | 긴급/디버깅용 수동 배치 | `nodeName` 직접 지정 |
| **Labels & Selectors** | 오브젝트 분류/필터링 | 키-값 쌍, `kubectl --selector` |
| **Taints & Tolerations** | 원치 않는 Pod 차단 | 노드가 Pod을 밀어냄 |
| **Node Selectors** | 단순 노드 선택 | `nodeSelector: key=value` |
| **Node Affinity** | 복잡한 노드 선택 | `In`, `NotIn`, `Exists` 연산자 |
| **Resource Requests** | 최소 리소스 보장 | 스케줄러의 노드 선택 기준 |
| **Resource Limits** | 최대 리소스 제한 | CPU=throttle, Memory=OOMKill |
| **LimitRange** | NS 기본 리소스 설정 | 새 Pod에 자동 적용 |
| **ResourceQuota** | NS 총량 제한 | 전체 리소스 사용량 제어 |
| **DaemonSet** | 노드당 Pod 1개 | 모니터링, 로그, 네트워크 에이전트 |
| **Static Pods** | kubelet 직접 관리 | Control Plane 부트스트랩 |
| **Multiple Schedulers** | 워크로드별 스케줄링 | `schedulerName` 필드 |
| **Scheduler Profiles** | 하나의 바이너리, 다중 전략 | K8s 1.18+ |

---

## 🦥 마무리

K8s 스케줄링은 결국 하나의 질문으로 귀결돼요: **"이 Pod을 어떤 노드에 올릴 것인가?"**

대부분의 경우 기본 스케줄러가 알아서 잘 처리해줘요. 하지만 프로덕션 환경에서는 노드 전용화, 리소스 관리, GPU 워크로드 분리 같은 요구사항이 생기기 마련이고, 그때 이 포스트에서 다룬 기법들이 진가를 발휘해요.

CKA 시험에서는 특히 Taints/Tolerations, Node Affinity, Resource Limits, Static Pods가 자주 출제되니까 꼭 실습해보는 걸 추천해요!
