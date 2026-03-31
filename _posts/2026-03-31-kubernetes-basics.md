---
title: "[Kubernetes] K8s 기본 구성 완벽 정리"
excerpt: "Kubernetes 클러스터의 구조와 핵심 오브젝트를 한눈에 정리해보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Container, Orchestration, DevOps]

permalink: /kubernetes/kubernetes-basics/

toc: true
toc_sticky: true

date: 2026-03-31
last_modified_at: 2026-03-31
---

## 🦥 Kubernetes란?

**Kubernetes**(줄여서 K8s)는 Google이 내부에서 쓰던 컨테이너 오케스트레이션 시스템 *Borg*를 기반으로 만든 오픈소스 프로젝트예요. 컨테이너화된 앱의 배포, 확장, 관리를 자동화해주는 플랫폼이고, 현재는 [CNCF](https://www.cncf.io/)에서 관리하고 있어요.

> Docker로 컨테이너를 만들 수 있지만, 수백 개의 컨테이너를 효율적으로 운영하려면 **오케스트레이션 도구**가 필요해요. 그 역할을 K8s가 담당하고 있죠!

---

## 🦥 클러스터 구조

K8s 클러스터는 크게 **Control Plane(마스터)**과 **Worker Node**로 나뉘어요. Control Plane이 클러스터 전체를 관리하는 두뇌 역할을 하고, Worker Node는 실제 앱이 돌아가는 곳이에요.

아래 그림으로 전체 구조를 한눈에 살펴볼게요.

![Kubernetes 클러스터 아키텍처](/assets/images/posts_img/kubernetes-basics/k8s-architecture.png)

### Control Plane 구성요소

| 구성요소 | 역할 |
|---|---|
| `kube-apiserver` | 클러스터의 관문, 모든 요청을 처리하는 REST API 서버 |
| `etcd` | 클러스터 상태 정보를 저장하는 분산 키-값 저장소 |
| `kube-scheduler` | 새로운 Pod를 어떤 노드에 배치할지 결정 |
| `kube-controller-manager` | 클러스터 상태를 desired state로 유지하는 컨트롤러 집합 |

**`kube-apiserver`** 는 `kubectl` 명령어가 통신하는 대상이에요. 인증, 인가, 어드미션 컨트롤을 담당하고, 클러스터 내부의 모든 컴포넌트도 이 서버를 거쳐서 통신해요.

```bash
# kubectl로 API Server와 통신하는 예시
kubectl get pods -n default
kubectl describe node worker-01
```

**`etcd`** 는 클러스터의 *"단일 진실 공급원(Single Source of Truth)"* 역할을 해요. 고가용성을 위해 보통 3개 이상의 인스턴스로 구성하고, Raft 합의 알고리즘으로 데이터 일관성을 보장하죠.

**`kube-scheduler`** 는 CPU/메모리 상태, `affinity` / `anti-affinity`, `taint` / `toleration` 등을 종합적으로 고려해서 최적의 노드를 골라줘요.

**`kube-controller-manager`** 에는 ReplicaSet Controller(Pod 수 유지), Node Controller(노드 상태 감시), Job Controller(일회성 작업 관리) 등이 포함되어 있어요.

---

### Worker Node 구성요소

| 구성요소 | 역할 |
|---|---|
| `kubelet` | 노드 에이전트, Pod 스펙대로 컨테이너 관리 |
| `kube-proxy` | 네트워크 규칙 관리, Service 트래픽 라우팅 |
| `Container Runtime` | 실제 컨테이너 실행 (`containerd`, `CRI-O`) |

**`kubelet`** 은 API Server에서 PodSpec을 받아 컨테이너를 관리하고, 헬스체크 결과를 주기적으로 보고해요.

**`kube-proxy`** 는 `iptables` 또는 `IPVS` 모드로 동작하면서, Service 트래픽을 적절한 Pod로 라우팅해줘요.

**`Container Runtime`** 은 containerd가 사실상 표준이에요. Docker는 K8s `1.24`부터 직접 지원이 중단되었거든요.

---

## 🦥 핵심 오브젝트

**Pod**는 K8s의 최소 배포 단위예요. 같은 Pod 내 컨테이너끼리 네트워크와 스토리지를 공유하고, 사이드카 패턴 같은 멀티 컨테이너 구성도 많이 쓰여요.

**Deployment**는 실무에서 가장 많이 쓰는 오브젝트예요. 롤링 업데이트와 롤백을 지원하고, 내부적으로 ReplicaSet을 통해 Pod 수를 유지해줘요.

```yaml
# Deployment 예시
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Service**는 Pod 집합에 안정적인 엔드포인트를 제공해요. Pod IP가 바뀌어도 걱정 없죠!

| 타입 | 용도 |
|---|---|
| `ClusterIP` | 클러스터 내부 통신 (기본값) |
| `NodePort` | 외부에서 노드 포트로 접근 |
| `LoadBalancer` | 클라우드 로드밸런서 연동 |

**ConfigMap과 Secret**은 설정 정보를 Pod와 분리해서 관리할 수 있게 해줘요. 이미지 변경 없이 환경별 설정을 유연하게 교체할 수 있어서 정말 편리해요.

**Namespace**는 클러스터를 논리적으로 분리하는 단위예요. 팀별, 환경별(`dev`/`staging`/`prod`)로 격리된 공간을 만들고 리소스 쿼터도 적용할 수 있어요.

---

## 🦥 동작 원리: 선언적 모델

K8s의 가장 큰 특징은 **선언형(declarative)** 방식이에요.

> *"Pod 3개를 실행해라"* 가 아니라 *"Pod 3개가 실행 중인 상태를 원한다"* 고 선언하면, 컨트롤러가 현재 상태와의 차이를 감지하고 자동으로 맞춰줘요.

이걸 **Reconciliation Loop**라고 부르는데, K8s의 모든 컨트롤러가 이 패턴으로 동작해요.

---

## 🦥 마무리

K8s의 핵심 철학은 **선언적 구성**과 **자동 복구(Self-healing)** 에 있어요. Pod가 죽으면 자동으로 다시 띄우고, 노드에 장애가 나면 다른 노드로 워크로드를 옮겨줘요. 이 아키텍처를 이해하는 게 K8s 활용의 첫걸음이고, 여기서부터 `Ingress`, `PV/PVC`, `Helm`, 모니터링 등으로 학습을 넓혀가면 돼요!
