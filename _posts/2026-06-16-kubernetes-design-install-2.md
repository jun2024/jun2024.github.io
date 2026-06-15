---
title: "[Kubernetes] K8s 클러스터 설계 & 설치 (2) — 고가용성(HA) 구성과 ETCD in HA"
excerpt: "프로덕션 K8s 클러스터의 필수! Control Plane HA 구성, kube-apiserver 로드밸런싱, ETCD HA 토폴로지와 Raft 합의 알고리즘까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, HA, High Availability, ETCD, Raft, CKA, DevOps]

permalink: /kubernetes/kubernetes-design-install-2/

toc: true
toc_sticky: true

date: 2026-06-16
last_modified_at: 2026-06-16
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Design and Install a Kubernetes Cluster 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [클러스터 설계 & 설치 (1)](/kubernetes/kubernetes-design-install-1/)에서 클러스터 설계와 인프라 선택을 다뤘어요!

---

## 🦥 왜 고가용성(HA)이 필요할까?

Master 노드가 1개뿐인 클러스터에서 Master가 다운되면 어떻게 될까요?

| Master 다운 시 | 영향 |
|---------------|------|
| 기존 Pod | **계속 동작** (kubelet이 독립적으로 관리) |
| 새 Pod 생성 | 불가 (apiserver 없음) |
| Pod 재스케줄링 | 불가 (scheduler 없음) |
| 자동 복구 | 불가 (controller-manager 없음) |
| kubectl 명령 | 불가 (apiserver 없음) |

기존 워크로드는 돌아가지만, **새로운 작업이나 장애 복구가 전혀 안 돼요**. 프로덕션에서는 이런 상황을 허용할 수 없어요.

> 💡 고가용성(HA) = Master 노드를 **여러 개** 두어 하나가 죽어도 클러스터가 정상 동작하도록 구성하는 거예요.

---

## 🦥 Control Plane HA 구성

### HA 구성 시 각 컴포넌트의 동작

Master가 여러 개일 때, 각 컴포넌트가 어떻게 동작하는지 이해해야 해요.

| 컴포넌트 | HA 동작 방식 | 설명 |
|---------|-------------|------|
| **kube-apiserver** | Active-Active | 모든 인스턴스가 동시에 요청 처리 |
| **kube-controller-manager** | Active-Standby | 1개만 활성, 나머지 대기 |
| **kube-scheduler** | Active-Standby | 1개만 활성, 나머지 대기 |
| **etcd** | 분산 합의 (Raft) | 모든 인스턴스가 데이터 공유 |

### kube-apiserver — Active-Active

apiserver는 **Stateless**하기 때문에 여러 인스턴스가 동시에 동작할 수 있어요. 앞에 **Load Balancer**를 두어 트래픽을 분산해요.

```
kubectl/사용자 요청
       │
  Load Balancer (Nginx, HAProxy 등)
       │
  ┌────┼────┐
  │    │    │
Master1 Master2 Master3
(apiserver) (apiserver) (apiserver)
```

| Load Balancer 옵션 | 설명 |
|-------------------|------|
| Nginx | 간단한 설정, 널리 사용 |
| HAProxy | 고성능, 상세한 헬스체크 |
| Cloud LB | 클라우드 환경 (AWS ALB/NLB, GCP LB) |
| kube-vip | K8s 네이티브 VIP 솔루션 |

```bash
# kubeadm HA 클러스터 초기화 시 LB 엔드포인트 지정
kubeadm init \
  --control-plane-endpoint "lb.example.com:6443" \
  --upload-certs
```

### kube-controller-manager — Active-Standby

controller-manager가 여러 개 동시에 동작하면 **같은 리소스를 중복으로 처리**할 수 있어요. 그래서 **Leader Election**을 사용해요.

```bash
# controller-manager 실행 옵션
kube-controller-manager \
  --leader-elect=true \                      # Leader Election 활성화 (기본값)
  --leader-elect-lease-duration=15s \        # 리스 유지 시간
  --leader-elect-renew-deadline=10s \        # 리스 갱신 기한
  --leader-elect-retry-period=2s             # 리스 획득 재시도 간격
```

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `--leader-elect` | Leader Election 사용 | true |
| `--leader-elect-lease-duration` | 리더의 임기(리스) 지속 시간 | 15s |
| `--leader-elect-renew-deadline` | 현재 리더가 리스를 갱신해야 하는 기한 | 10s |
| `--leader-elect-retry-period` | 대기 인스턴스의 리더 획득 재시도 간격 | 2s |

**동작 흐름**이에요.

| 단계 | 동작 |
|------|------|
| 1 | 모든 인스턴스가 시작되면 리더 선출 시도 |
| 2 | 먼저 Endpoint/Lease 리소스를 획득한 인스턴스가 리더 |
| 3 | 리더만 실제 컨트롤러 로직 실행 |
| 4 | 나머지는 대기 (Standby) |
| 5 | 리더가 다운되면 대기 인스턴스 중 하나가 새 리더 |

### kube-scheduler — Active-Standby

scheduler도 controller-manager와 동일하게 **Leader Election**으로 동작해요.

```bash
kube-scheduler \
  --leader-elect=true
```

---

## 🦥 ETCD in HA

### ETCD HA가 중요한 이유

ETCD는 클러스터의 **모든 상태 데이터**를 저장해요. ETCD가 유실되면 클러스터 전체를 잃는 거예요. HA 구성에서 ETCD는 가장 신중하게 설계해야 해요.

### ETCD HA 토폴로지

| 토폴로지 | 설명 | 장점 | 단점 |
|---------|------|------|------|
| **Stacked** | Master 노드 안에 ETCD 배치 | 구성 간단, 노드 수 적음 | Master 장애 = ETCD도 장애 |
| **External** | 별도 서버에 ETCD 배치 | ETCD 독립 관리, 장애 격리 | 서버 추가 비용 |

### Stacked 토폴로지

```
Master 1               Master 2               Master 3
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│ apiserver    │      │ apiserver    │      │ apiserver    │
│ scheduler    │      │ scheduler    │      │ scheduler    │
│ controller   │      │ controller   │      │ controller   │
│ etcd         │      │ etcd         │      │ etcd         │
└──────────────┘      └──────────────┘      └──────────────┘
```

### External 토폴로지

```
Master 1          Master 2          Master 3
┌────────────┐   ┌────────────┐   ┌────────────┐
│ apiserver  │   │ apiserver  │   │ apiserver  │
│ scheduler  │   │ scheduler  │   │ scheduler  │
│ controller │   │ controller │   │ controller │
└────────────┘   └────────────┘   └────────────┘

ETCD 1            ETCD 2            ETCD 3
┌────────────┐   ┌────────────┐   ┌────────────┐
│ etcd       │   │ etcd       │   │ etcd       │
└────────────┘   └────────────┘   └────────────┘
```

> 💡 **CKA 포인트**: kubeadm 기본 설치는 **Stacked** 토폴로지예요. External은 대규모 프로덕션에서 사용해요.

---

## 🦥 ETCD와 Raft 합의 알고리즘

### 분산 시스템의 문제

ETCD 인스턴스가 3개인데 동시에 다른 값을 쓰면 어떻게 될까요? **모든 인스턴스가 동일한 데이터를 유지**해야 해요. 이걸 **합의(Consensus)**라고 해요.

### Raft 알고리즘

ETCD는 **Raft** 합의 알고리즘을 사용해요.

| 역할 | 설명 |
|------|------|
| **Leader** | 모든 쓰기 요청을 처리, 다른 노드에 복제 |
| **Follower** | Leader의 데이터를 복제, 읽기 요청 처리 가능 |
| **Candidate** | Leader 선출 과정에서의 임시 상태 |

### Leader 선출 과정

| 단계 | 동작 |
|------|------|
| 1 | 클러스터 시작 시 모든 노드가 Follower |
| 2 | 타이머가 만료되면 Candidate로 전환 |
| 3 | 다른 노드에 투표 요청 |
| 4 | 과반수 표를 받으면 Leader로 선출 |
| 5 | Leader가 주기적으로 Heartbeat 전송 |
| 6 | Heartbeat가 중단되면 새 선거 시작 |

### 쓰기 과정

```
클라이언트 → Leader에 쓰기 요청
  → Leader가 자신에 저장
  → 다른 Follower에 복제 요청
  → 과반수가 저장 완료하면 커밋
  → 클라이언트에 성공 응답
```

> 💡 **핵심**: 쓰기는 **과반수(Quorum)**가 동의해야 커밋돼요. 과반수 = (N/2) + 1

### Quorum (정족수)

| ETCD 노드 수 | Quorum | 허용 장애 수 | 추천 여부 |
|-------------|--------|------------|----------|
| 1 | 1 | 0 | 학습용만 |
| 2 | 2 | 0 | 비추천 (1개 죽으면 쓰기 불가) |
| **3** | **2** | **1** | **최소 HA 구성** |
| 4 | 3 | 1 | 비추천 (3과 동일한 내결함성) |
| **5** | **3** | **2** | **권장 프로덕션** |
| 6 | 4 | 2 | 비추천 (5와 동일한 내결함성) |
| **7** | **4** | **3** | **대규모** |

> 💡 **핵심 공식**: Quorum = (N/2) + 1 (소수점 버림). 항상 **홀수**로 구성해야 효율적이에요. 짝수는 노드를 하나 더 쓰면서 내결함성은 같아요.

### 왜 짝수는 비효율적인가?

| 비교 | 3노드 | 4노드 |
|------|-------|-------|
| Quorum | 2 | 3 |
| 허용 장애 | 1 | 1 |
| 결론 | - | 노드 1개 더 쓰는데 내결함성 동일 |

| 비교 | 5노드 | 6노드 |
|------|-------|-------|
| Quorum | 3 | 4 |
| 허용 장애 | 2 | 2 |
| 결론 | - | 노드 1개 더 쓰는데 내결함성 동일 |

---

## 🦥 ETCD 클러스터 설정

### Stacked ETCD (kubeadm)

kubeadm으로 HA 클러스터를 구성할 때 ETCD는 자동으로 Static Pod으로 배포돼요.

```bash
# 첫 번째 Master 초기화
kubeadm init \
  --control-plane-endpoint "lb.example.com:6443" \
  --upload-certs

# 추가 Master 조인
kubeadm join lb.example.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash <hash> \
  --control-plane \
  --certificate-key <cert-key>
```

### External ETCD

apiserver가 External ETCD에 연결하도록 설정해요.

```yaml
# kube-apiserver 설정
spec:
  containers:
  - command:
    - kube-apiserver
    - --etcd-servers=https://etcd1:2379,https://etcd2:2379,https://etcd3:2379
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
```

### ETCD 클러스터 멤버 설정

각 ETCD 인스턴스는 다른 멤버를 알고 있어야 해요.

```bash
# ETCD 실행 옵션
etcd \
  --name etcd1 \
  --initial-cluster etcd1=https://10.0.0.1:2380,etcd2=https://10.0.0.2:2380,etcd3=https://10.0.0.3:2380 \
  --initial-cluster-state new \
  --listen-peer-urls https://10.0.0.1:2380 \
  --listen-client-urls https://10.0.0.1:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.1:2379 \
  --initial-advertise-peer-urls https://10.0.0.1:2380
```

| 포트 | 용도 |
|------|------|
| 2379 | 클라이언트 통신 (apiserver → etcd) |
| 2380 | 피어 통신 (etcd ↔ etcd) |

---

## 🦥 CKA 시험 대비 — HA 치트시트

```bash
# ETCD 클러스터 멤버 확인
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# ETCD 클러스터 상태 확인
ETCDCTL_API=3 etcdctl endpoint status --cluster \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-table

# Controller Manager의 현재 리더 확인
kubectl get endpoints kube-controller-manager -n kube-system -o yaml

# Scheduler의 현재 리더 확인
kubectl get endpoints kube-scheduler -n kube-system -o yaml
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **HA 필요성** | Master 단일 장애점 제거, 프로덕션 필수 |
| **apiserver HA** | Active-Active, Load Balancer로 분산 |
| **controller-manager/scheduler** | Active-Standby, Leader Election |
| **ETCD 토폴로지** | Stacked (Master 내부) vs External (별도 서버) |
| **Raft 알고리즘** | Leader 선출, 과반수 합의로 데이터 일관성 |
| **Quorum** | (N/2)+1, 항상 홀수 구성 (3, 5, 7) |
| **ETCD 포트** | 2379 (클라이언트), 2380 (피어) |
| **kubeadm HA** | `--control-plane-endpoint`로 LB 지정 |

이것으로 클러스터 설계 & 설치 섹션을 마무리할게요! 클러스터 설계부터 인프라 선택, HA 구성, ETCD Raft 합의까지 — CKA 시험에 필요한 핵심을 모두 다뤘어요. 🦥
