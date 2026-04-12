---
title: "[Kubernetes] K8s 모니터링 & 로깅 기본 개념 — CKA 핵심 정리"
excerpt: "Metrics Server, kubectl top, kubectl logs부터 클러스터 컴포넌트 모니터링까지 — CKA 시험에 나오는 모니터링/로깅 핵심 개념을 총정리해보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Monitoring, Logging, CKA, DevOps]

permalink: /kubernetes/kubernetes-monitoring-logging-basics/

toc: true
toc_sticky: true

date: 2026-04-12
last_modified_at: 2026-04-12
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Logging & Monitoring 섹션을 기반으로 정리한 내용이에요.
> 다음 포스트에서는 Prometheus + Grafana를 활용한 실무 모니터링 구성을 다뤄요!

---

## 🦥 왜 모니터링과 로깅이 필요할까?

K8s 클러스터를 운영하다 보면 이런 질문들이 생겨요.

- "노드 CPU가 몇 % 사용 중이지?"
- "이 Pod이 왜 OOMKilled 됐지?"
- "애플리케이션 에러 로그를 어떻게 확인하지?"

K8s 자체는 빌트인 모니터링 대시보드를 제공하지 않아요. 대신 **Metrics Server**라는 경량 컴포넌트와 `kubectl` 명령어를 통해 기본적인 리소스 사용량과 로그를 확인할 수 있어요. CKA 시험에서도 이 부분이 자주 출제되니 확실하게 정리해볼게요!

---

## 🦥 클러스터 모니터링의 전체 구조

K8s 모니터링은 크게 **노드 레벨**과 **Pod 레벨**로 나뉘어요.

```
┌─────────────────────────────────────────────────┐
│               Kubernetes Cluster                │
│                                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │        │
│  │┌───────┐│  │┌───────┐│  │┌───────┐│        │
│  ││kubelet ││  ││kubelet ││  ││kubelet ││        │
│  ││+cAdvis││  ││+cAdvis││  ││+cAdvis││        │
│  │└───┬───┘│  │└───┬───┘│  │└───┬───┘│        │
│  └────┼────┘  └────┼────┘  └────┼────┘        │
│       └────────────┼───────────┘               │
│                    ▼                            │
│          ┌─────────────────┐                   │
│          │  Metrics Server  │                   │
│          │  (In-Memory)     │                   │
│          └────────┬────────┘                   │
│                   ▼                            │
│          ┌─────────────────┐                   │
│          │  kube-apiserver  │                   │
│          └─────────────────┘                   │
│                   ▼                            │
│           kubectl top node                     │
│           kubectl top pod                      │
└─────────────────────────────────────────────────┘
```

### 모니터링 레벨별 주요 지표

| 레벨 | 주요 지표 | 확인 목적 |
|------|----------|----------|
| **노드** | CPU 사용률, 메모리 사용률, 디스크, 네트워크 | 노드 과부하 감지, 스케일링 판단 |
| **Pod** | CPU/메모리 Request vs 실사용량 | OOM 방지, 리소스 튜닝 |
| **컨테이너** | 개별 컨테이너 리소스 사용량 | 멀티 컨테이너 Pod 디버깅 |

---

## 🦥 Metrics Server 이해하기

### Metrics Server란?

과거에는 **Heapster**라는 모니터링 도구를 사용했는데, 현재는 deprecated 되었고 경량화된 **Metrics Server**가 대체하고 있어요.

Metrics Server의 핵심 특징은 이래요.

| 특징 | 설명 |
|------|------|
| **In-Memory** | 데이터를 디스크에 저장하지 않고 메모리에만 보관해요 |
| **실시간** | 현재 시점의 리소스 사용량만 확인 가능해요 |
| **경량** | 클러스터에 최소한의 리소스만 사용해요 |
| **과거 데이터 없음** | 히스토리 조회가 불가능해요 (Prometheus 필요) |

> 💡 **CKA 포인트**: Metrics Server는 과거 데이터를 저장하지 않아요! 과거 데이터가 필요하면 Prometheus 같은 외부 모니터링 솔루션이 필요해요.

### 메트릭 수집 흐름

각 노드의 **kubelet** 안에는 **cAdvisor (Container Advisor)**라는 컴포넌트가 내장되어 있어요. cAdvisor가 컨테이너의 CPU, 메모리 등의 메트릭을 수집하고, 이것을 Metrics Server가 모아서 kube-apiserver를 통해 제공하는 구조예요.

```
Container → cAdvisor (kubelet 내장) → Metrics Server → kube-apiserver → kubectl top
```

### Metrics Server 설치

대부분의 관리형 K8s(EKS, GKE, AKS)에서는 기본 설치되어 있지만, kubeadm이나 minikube에서는 직접 설치해야 해요.

```bash
# minikube에서 활성화
minikube addons enable metrics-server

# 일반 클러스터에서 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

설치 후 바로 메트릭이 수집되지 않아요. **1~2분 정도** 기다린 후 데이터가 쌓이면 `kubectl top` 명령어를 사용할 수 있어요.

```bash
# Metrics Server 동작 확인
kubectl get pods -n kube-system | grep metrics-server
```

---

## 🦥 kubectl top — 리소스 사용량 확인

Metrics Server가 설치되면 `kubectl top` 명령어로 리소스 사용량을 확인할 수 있어요. CKA 시험에서 매우 자주 나오는 명령어예요!

### 노드 리소스 확인

```bash
kubectl top node
```

```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
controlplane   250m         12%    1024Mi          54%
node01         180m         9%     768Mi           40%
node02         320m         16%    1200Mi          63%
```

각 컬럼의 의미는 이래요.

| 컬럼 | 설명 |
|------|------|
| CPU(cores) | 현재 CPU 사용량 (밀리코어 단위, 1000m = 1 core) |
| CPU% | 노드 전체 CPU 대비 사용률 |
| MEMORY(bytes) | 현재 메모리 사용량 |
| MEMORY% | 노드 전체 메모리 대비 사용률 |

### Pod 리소스 확인

```bash
# 현재 네임스페이스의 Pod 리소스
kubectl top pod

# 특정 네임스페이스
kubectl top pod -n kube-system

# 전체 네임스페이스
kubectl top pod -A

# 컨테이너별 리소스 확인
kubectl top pod --containers
```

```
NAME                     CPU(cores)   MEMORY(bytes)
nginx-7f8c9b4d5c-abc12   5m           32Mi
webapp-5d8f7b9c4-def34   120m         256Mi
redis-6c9f8d7e5-ghi56    15m          64Mi
```

### CKA 시험 빈출 패턴

CKA 시험에서 자주 나오는 `kubectl top` 관련 문제 유형들이에요.

```bash
# CPU 사용량 기준 정렬
kubectl top pod --sort-by=cpu

# 메모리 사용량 기준 정렬
kubectl top pod --sort-by=memory

# 특정 레이블의 Pod만 조회
kubectl top pod -l app=webapp

# 가장 리소스를 많이 사용하는 노드 찾기
kubectl top node --sort-by=cpu
```

> 💡 **CKA 팁**: "가장 CPU를 많이 사용하는 Pod을 찾아라" 같은 문제가 자주 나와요. `--sort-by=cpu` 옵션을 기억하세요!

---

## 🦥 클러스터 컴포넌트 모니터링

K8s 클러스터의 핵심 컴포넌트들도 모니터링해야 해요. 컴포넌트가 정상 동작하지 않으면 전체 클러스터에 영향을 줄 수 있어요.

### 핵심 컴포넌트 상태 확인

```bash
# 컴포넌트 상태 확인 (deprecated되었지만 참고용)
kubectl get componentstatuses   # 또는 kubectl get cs

# kube-system 네임스페이스의 Pod 확인
kubectl get pods -n kube-system

# 개별 컴포넌트 로그 확인
kubectl logs kube-apiserver-controlplane -n kube-system
kubectl logs kube-scheduler-controlplane -n kube-system
kubectl logs kube-controller-manager-controlplane -n kube-system
```

### 주요 컴포넌트별 확인 포인트

| 컴포넌트 | 확인 방법 | 주요 이슈 |
|----------|----------|----------|
| **kube-apiserver** | `kubectl logs`, `/healthz` 엔드포인트 | 인증/인가 오류, etcd 연결 실패 |
| **kube-scheduler** | `kubectl logs` | Pod pending 상태 장기화 |
| **kube-controller-manager** | `kubectl logs` | ReplicaSet 미동작, 노드 감지 실패 |
| **etcd** | `kubectl logs`, `etcdctl endpoint health` | 클러스터 데이터 손상, 리더 선출 실패 |
| **kubelet** | `systemctl status kubelet`, `journalctl -u kubelet` | 노드 NotReady, Pod 생성 실패 |
| **kube-proxy** | `kubectl logs`, DaemonSet 상태 확인 | 서비스 통신 불가 |

> 💡 kubelet은 Static Pod이 아니라 systemd 서비스로 동작하기 때문에 `journalctl`로 로그를 확인해야 해요!

---

## 🦥 kubectl logs — 애플리케이션 로그 관리

### 기본 로그 확인

```bash
# Pod 로그 확인
kubectl logs <pod-name>

# 실시간 로그 스트리밍 (-f: follow)
kubectl logs -f <pod-name>

# 마지막 N줄만 확인
kubectl logs --tail=100 <pod-name>

# 최근 1시간 로그만 확인
kubectl logs --since=1h <pod-name>

# 이전 컨테이너 로그 (재시작된 경우)
kubectl logs --previous <pod-name>
```

### 멀티 컨테이너 Pod 로그

하나의 Pod에 여러 컨테이너가 있을 때는 반드시 **컨테이너 이름을 명시**해야 해요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
  - name: app
    image: nginx
  - name: sidecar-logger
    image: busybox
    command: ['sh', '-c', 'while true; do echo "$(date) - sidecar log"; sleep 5; done']
```

```bash
# 컨테이너 지정하지 않으면 에러 발생
kubectl logs multi-container-pod
# error: a container name must be specified for pod multi-container-pod

# 컨테이너 이름을 명시해야 해요
kubectl logs multi-container-pod -c app
kubectl logs multi-container-pod -c sidecar-logger

# 실시간 스트리밍 + 컨테이너 지정
kubectl logs -f multi-container-pod -c sidecar-logger
```

> 💡 **CKA 포인트**: 멀티 컨테이너 Pod에서 로그를 확인할 때 `-c <container-name>` 옵션을 빠뜨리면 에러가 나요. 시험에서 자주 출제되는 패턴이에요!

### Init Container 로그

Init Container의 로그도 같은 방식으로 확인할 수 있어요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init-db-check
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb-service; do echo "waiting for DB..."; sleep 2; done']
  containers:
  - name: app
    image: nginx
```

```bash
# Init Container 로그 확인
kubectl logs init-demo -c init-db-check
```

---

## 🦥 로그 아키텍처 이해하기

K8s에서 컨테이너 로그가 어떻게 관리되는지 아키텍처를 이해하면 트러블슈팅에 큰 도움이 돼요.

### 컨테이너 로그 저장 위치

컨테이너의 **stdout/stderr** 출력은 노드의 파일시스템에 저장돼요.

```
노드 파일시스템 경로:
/var/log/containers/<pod-name>_<namespace>_<container-name>-<container-id>.log
  → 심볼릭 링크 →
/var/log/pods/<pod-uid>/<container-name>/0.log
  → 심볼릭 링크 →
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

### 로그 로테이션

K8s는 기본적으로 로그 파일이 무한히 커지는 것을 방지해요.

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `containerLogMaxSize` | 10Mi | 단일 로그 파일 최대 크기 |
| `containerLogMaxFiles` | 5 | 보관할 로그 파일 개수 |

> 💡 Pod이 삭제되면 해당 로그도 함께 삭제돼요! 그래서 실무에서는 외부 로깅 시스템(EFK, Loki 등)으로 로그를 수집하는 게 필수예요.

### K8s 로깅 패턴 3가지

K8s에서 로그를 수집하는 대표적인 패턴은 3가지가 있어요.

**1. 노드 레벨 에이전트 (권장)**
```
각 노드에 DaemonSet으로 로그 수집 에이전트를 배포
→ Fluentd, Fluent Bit, Filebeat 등
→ 중앙 로그 저장소로 전송 (Elasticsearch, Loki 등)
```

**2. 사이드카 컨테이너**
```
Pod 내부에 로그 수집 전용 사이드카 컨테이너를 추가
→ 애플리케이션 로그 파일을 읽어서 stdout으로 출력
→ 또는 직접 외부 로그 시스템으로 전송
```

**3. 애플리케이션 직접 전송**
```
애플리케이션 코드에서 직접 로그 시스템으로 전송
→ 가장 유연하지만 애플리케이션 수정 필요
```

---

## 🦥 실무에서 자주 쓰는 디버깅 명령어 모음

CKA 시험과 실무 모두에서 유용한 명령어들을 정리했어요.

### Pod 상태 확인

```bash
# Pod 이벤트 확인 (스케줄링 실패, 이미지 풀 에러 등)
kubectl describe pod <pod-name>

# Pod 상태가 이상한 경우
kubectl get pod <pod-name> -o wide

# 특정 네임스페이스의 모든 이벤트 확인
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### 리소스 관련 디버깅

```bash
# Pod이 Pending 상태일 때 — 리소스 부족 확인
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# 노드별 Pod 수 확인
kubectl get pods -A -o wide | grep <node-name> | wc -l

# 리소스 요청량 vs 실사용량 비교
kubectl top pod <pod-name>
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'
```

### 빠른 트러블슈팅 플로우

```
Pod 문제 발생
  ├─ kubectl get pod (상태 확인)
  │   ├─ Pending → kubectl describe pod (이벤트 확인)
  │   │   ├─ Insufficient CPU/Memory → kubectl top node
  │   │   └─ No nodes match → Taint/Affinity 확인
  │   ├─ CrashLoopBackOff → kubectl logs --previous
  │   ├─ ImagePullBackOff → 이미지 이름/레지스트리 확인
  │   └─ Running (but not working) → kubectl logs -f
  └─ kubectl describe pod (전체 이벤트/상태 확인)
```

---

## 🦥 모니터링 솔루션 비교 (Overview)

K8s 생태계에서 사용할 수 있는 모니터링/로깅 솔루션들을 간단히 비교해볼게요. 다음 포스트에서 Prometheus + Grafana를 상세하게 다룰 예정이에요!

### 모니터링 솔루션

| 솔루션 | 특징 | 적합한 환경 |
|--------|------|------------|
| **Metrics Server** | 경량, 실시간, 과거 데이터 없음 | CKA 시험, 기본 확인용 |
| **Prometheus + Grafana** | 오픈소스, 시계열 DB, 알림 | 대부분의 프로덕션 환경 |
| **Datadog** | SaaS, 올인원, 비용 발생 | 관리 부담 최소화 원하는 팀 |
| **Dynatrace** | AI 기반 자동 분석, 비용 발생 | 엔터프라이즈 환경 |

### 로깅 솔루션

| 솔루션 | 특징 | 적합한 환경 |
|--------|------|------------|
| **kubectl logs** | 빌트인, 실시간, 개별 Pod만 | 간단한 디버깅, CKA 시험 |
| **EFK Stack** | Elasticsearch + Fluentd + Kibana | 대규모, 검색 중심 |
| **Loki + Grafana** | 경량, 라벨 기반, Prometheus와 통합 | 비용 효율적 환경 |
| **CloudWatch / Stackdriver** | 클라우드 네이티브 | AWS/GCP 환경 |

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Metrics Server** | In-Memory 경량 모니터링, cAdvisor → Metrics Server → API Server |
| **kubectl top** | 노드/Pod 리소스 실시간 확인, `--sort-by` 옵션 기억! |
| **kubectl logs** | `-f` (실시간), `-c` (멀티 컨테이너), `--previous` (이전 컨테이너) |
| **로그 저장** | stdout/stderr → 노드 파일시스템, Pod 삭제 시 함께 삭제 |
| **로깅 패턴** | 노드 에이전트(DaemonSet) 방식이 가장 권장 |
| **외부 솔루션** | 실무에서는 Prometheus + Grafana / EFK or Loki 조합 필수 |

다음 포스트에서는 **Prometheus + Grafana를 활용한 실무 모니터링 구성**을 다뤄볼게요. Helm으로 설치하고, 대시보드 구성까지 실습해볼 거예요! 🦥
