---
title: "[Kubernetes] K8s 클러스터 설계 & 설치 (1) — 클러스터 설계와 인프라 선택 가이드"
excerpt: "프로덕션 K8s 클러스터를 어떻게 설계할까? 목적별 클러스터 구성, 노드 사이징, 온프레미스 vs 클라우드 인프라 선택까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Cluster Design, Infrastructure, kubeadm, CKA, DevOps]

permalink: /kubernetes/kubernetes-design-install-1/

toc: true
toc_sticky: true

date: 2026-06-15
last_modified_at: 2026-06-15
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Design and Install a Kubernetes Cluster 섹션을 기반으로 정리한 내용이에요.

---

## 🦥 클러스터를 설계하기 전에

K8s 클러스터를 만들기 전에 먼저 **목적**과 **요구사항**을 파악해야 해요. 목적에 따라 구성이 완전히 달라져요.

| 질문 | 고려 사항 |
|------|----------|
| 목적이 뭔가요? | 학습용? 개발/테스트? 프로덕션? |
| 얼마나 큰가요? | 사용자 수, 워크로드 유형, 트래픽 규모 |
| 어디에 호스팅하나요? | 클라우드? 온프레미스? |
| 워크로드 특성은? | 웹 앱? 빅데이터? ML/AI? |

---

## 🦥 목적별 클러스터 구성

### 학습용 (Education)

| 항목 | 구성 |
|------|------|
| 도구 | Minikube, kind, kubeadm (단일 노드) |
| 노드 수 | 1개 |
| 리소스 | 최소 2 CPU, 2GB RAM |
| 특징 | 로컬 머신에서 빠르게 실행 |

### 개발/테스트 (Dev/Test)

| 항목 | 구성 |
|------|------|
| 도구 | kubeadm (멀티 노드), 관리형 K8s |
| 노드 수 | Master 1개 + Worker 1~2개 |
| 리소스 | 노드당 4 CPU, 8GB RAM |
| 특징 | 프로덕션과 유사한 환경 |

### 프로덕션 (Production)

| 항목 | 구성 |
|------|------|
| 도구 | kubeadm, kops, 관리형 K8s (EKS, GKE, AKS) |
| 노드 수 | Master 3개(HA) + Worker 다수 |
| 리소스 | 워크로드에 따라 산정 |
| 특징 | 고가용성, 모니터링, 백업 필수 |

---

## 🦥 클러스터 노드 설계

### Control Plane (Master) 노드

| 구성 요소 | 역할 | 리소스 영향 |
|----------|------|-----------|
| kube-apiserver | API 요청 처리 | CPU, 메모리 |
| etcd | 클러스터 상태 저장 | 디스크 I/O, 메모리 |
| kube-scheduler | Pod 스케줄링 | CPU |
| kube-controller-manager | 컨트롤러 실행 | CPU, 메모리 |

### Master 노드 사이징 가이드

| 노드 수 (Worker) | 권장 Master 스펙 |
|------------------|-----------------|
| ~5개 | 1 CPU, 4GB RAM |
| ~10개 | 2 CPU, 8GB RAM |
| ~100개 | 4 CPU, 16GB RAM |
| ~500개 | 8 CPU, 32GB RAM |

### Worker 노드 고려사항

| 고려 사항 | 설명 |
|----------|------|
| 워크로드 유형 | CPU 집약적? 메모리 집약적? GPU 필요? |
| Pod 수 | 노드당 기본 최대 110개 Pod |
| 디스크 | 로컬 스토리지 필요 여부 |
| 네트워크 | 고대역폭 필요 여부 |

### 큰 노드 vs 많은 노드

| 전략 | 장점 | 단점 |
|------|------|------|
| 적은 수의 큰 노드 | 오버헤드 적음, 관리 간편 | 장애 시 영향 범위 큼 |
| 많은 수의 작은 노드 | 장애 격리 우수, 세밀한 확장 | 관리 복잡, 리소스 오버헤드 |

> 💡 **실무 팁**: 처음에는 중간 크기 노드로 시작하고, 워크로드 패턴을 파악한 후 최적화하는 게 좋아요.

---

## 🦥 Master와 Worker 분리

### 프로덕션에서는 반드시 분리

| 설정 | 학습/개발 | 프로덕션 |
|------|----------|---------|
| Master에서 워크로드 실행 | 가능 (Taint 제거) | 비추천 |
| Master-Worker 분리 | 선택 | 필수 |
| 전용 etcd 노드 | 불필요 | 대규모 시 권장 |

```bash
# Master 노드의 Taint 확인
kubectl describe node master-node | grep Taint

# 학습용: Master에서도 Pod 스케줄 허용
kubectl taint nodes master-node node-role.kubernetes.io/control-plane:NoSchedule-

# 프로덕션: Taint 유지 (기본값)
```

> 💡 kubeadm으로 설치하면 Master 노드에 `NoSchedule` Taint가 자동 설정돼요. 프로덕션에서는 이걸 제거하지 마세요.

---

## 🦥 인프라 선택 — 어디에 설치할까?

### Linux 선택

K8s는 **Linux** 위에서 동작해요. Control Plane 컴포넌트는 반드시 Linux여야 해요.

| OS | 지원 여부 | 비고 |
|----|----------|------|
| Ubuntu | Control Plane + Worker | 가장 널리 사용 |
| CentOS/RHEL | Control Plane + Worker | 엔터프라이즈 환경 |
| Windows | Worker만 가능 | Windows 컨테이너 전용 |

### 배포 방식

| 방식 | 설명 | 사용 사례 |
|------|------|----------|
| **Turnkey Solutions** | 직접 VM 프로비저닝 + K8s 설치 | 온프레미스, 세밀한 제어 필요 시 |
| **Managed Services** | 클라우드가 Control Plane 관리 | 빠른 시작, 운영 부담 최소화 |

### Turnkey Solutions

직접 인프라를 준비하고 K8s를 설치하는 방식이에요.

| 도구 | 설명 |
|------|------|
| **kubeadm** | K8s 공식 설치 도구, CKA 시험 범위 |
| **kops** | AWS에서 클러스터 프로비저닝 + 설치 자동화 |
| **kubespray** | Ansible 기반 설치 자동화 |

```bash
# kubeadm으로 클러스터 초기화 (Master)
kubeadm init --pod-network-cidr=10.244.0.0/16

# Worker 노드 조인
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

### Managed Kubernetes Services

클라우드 프로바이더가 Control Plane을 관리해줘요.

| 서비스 | 클라우드 | 특징 |
|--------|---------|------|
| **EKS** | AWS | 가장 많은 사용자, Fargate 지원 |
| **GKE** | GCP | K8s 원조, Auto-pilot 모드 |
| **AKS** | Azure | Active Directory 통합 |

| 구분 | Turnkey | Managed |
|------|---------|---------|
| Control Plane 관리 | 직접 | 클라우드 |
| Master 노드 비용 | 직접 부담 | 포함 (또는 무료) |
| 업그레이드 | 수동 | 간편 (클릭/명령) |
| 커스터마이징 | 자유로움 | 일부 제한 |
| etcd 관리 | 직접 | 클라우드 |

> 💡 **CKA 시험**에서는 `kubeadm`을 사용한 클러스터 설치가 출제 범위예요. Managed Service는 시험에 나오지 않아요.

---

## 🦥 네트워크 설계

### CIDR 대역 계획

클러스터의 네트워크 대역이 겹치지 않도록 미리 계획해야 해요.

| 대역 | 용도 | 예시 |
|------|------|------|
| 노드 네트워크 | 노드 간 통신 | 192.168.0.0/16 |
| Pod 네트워크 | Pod IP 할당 | 10.244.0.0/16 |
| Service 네트워크 | Service ClusterIP 할당 | 10.96.0.0/12 |

> ⚠️ **주의**: 세 대역이 서로 겹치면 안 돼요. 기존 사내 네트워크와도 겹치지 않도록 확인하세요.

### CNI 플러그인 선택

| CNI | 특징 | 추천 환경 |
|-----|------|----------|
| Flannel | 간단, 설정 쉬움 | 소규모, 학습용 |
| Calico | NetworkPolicy 지원, BGP | 프로덕션 |
| Weave | 간편한 설치, Overlay | 중소규모 |
| Cilium | eBPF 기반, 고성능 | 대규모, 보안 중시 |

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **목적별 구성** | 학습(1노드) → 개발(Master 1 + Worker 2) → 프로덕션(HA) |
| **노드 사이징** | 워크로드 규모에 따라 Master/Worker 스펙 결정 |
| **Master-Worker 분리** | 프로덕션에서는 반드시 분리, Taint 유지 |
| **Turnkey vs Managed** | 직접 설치(kubeadm) vs 클라우드 관리(EKS, GKE, AKS) |
| **kubeadm** | CKA 시험 범위, 공식 설치 도구 |
| **네트워크 설계** | Node/Pod/Service CIDR 대역 겹침 방지 |

다음 포스트에서는 **고가용성(HA) 구성과 ETCD in HA**를 다룰게요! 🦥
