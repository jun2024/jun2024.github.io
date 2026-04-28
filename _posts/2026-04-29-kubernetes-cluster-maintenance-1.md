---
title: "[Kubernetes] K8s 클러스터 유지보수 (1) — OS Upgrades와 Cluster Upgrade"
excerpt: "노드 OS 업그레이드 시 drain/cordon/uncordon 사용법부터 K8s 버전 관리, kubeadm을 이용한 클러스터 업그레이드 전체 프로세스까지 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, ClusterMaintenance, Upgrade, CKA, DevOps]

permalink: /kubernetes/kubernetes-cluster-maintenance-1/

toc: true
toc_sticky: true

date: 2026-04-29
last_modified_at: 2026-04-29
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Cluster Maintenance 섹션을 기반으로 정리한 내용이에요.
> 다음 포스트에서는 Backup & Restore, ETCDCTL을 다뤄요!

---

## 🦥 OS Upgrades — 노드 유지보수

### 노드를 내려야 하는 상황

클러스터를 운영하다 보면 노드의 OS 패치, 커널 업그레이드, 하드웨어 교체 등으로 노드를 내려야 하는 경우가 있어요. 이때 해당 노드의 Pod들을 안전하게 이동시켜야 해요.

### 노드가 갑자기 다운되면?

노드가 다운되면 kube-controller-manager는 **pod-eviction-timeout** (기본 5분) 동안 기다려요. 그 안에 노드가 복구되면 Pod이 그대로 유지되고, 복구되지 않으면 Pod이 다른 노드에 재스케줄링돼요.

| 상황 | 결과 |
|------|------|
| 5분 내 노드 복구 | 기존 Pod 유지 |
| 5분 후에도 미복구 | ReplicaSet/Deployment의 Pod만 다른 노드에 재생성 |
| 단독 Pod (ReplicaSet 없음) | **영구 유실** — 재생성 안 됨 |

> 💡 그래서 **계획된 유지보수**에서는 반드시 `kubectl drain`을 사용해서 Pod을 안전하게 이동시킨 후 노드를 내려야 해요.

### kubectl drain — 안전한 노드 비우기

`drain`은 노드의 모든 Pod을 다른 노드로 이동(evict)시키고, 해당 노드를 **SchedulingDisabled** 상태로 만들어요.

```bash
# 노드 비우기 (Pod을 안전하게 다른 노드로 이동)
kubectl drain node01

# DaemonSet Pod 무시 (DaemonSet은 모든 노드에 있어야 하므로)
kubectl drain node01 --ignore-daemonsets

# 단독 Pod(ReplicaSet 없는)도 강제 삭제
kubectl drain node01 --ignore-daemonsets --force

# 로컬 데이터가 있는 Pod도 삭제 허용
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
```

`drain` 실행 후 노드 상태를 확인하면 이래요.

```bash
kubectl get nodes
```

```
NAME           STATUS                     ROLES           AGE
controlplane   Ready                      control-plane   30d
node01         Ready,SchedulingDisabled   <none>          30d
node02         Ready                      <none>          30d
```

> 💡 **CKA 빈출**: `kubectl drain`이 실패하는 경우가 있어요. DaemonSet Pod이 있으면 `--ignore-daemonsets`, 단독 Pod이 있으면 `--force`를 추가해야 해요. 시험에서 에러 메시지를 보고 적절한 옵션을 추가하는 문제가 자주 나와요.

### kubectl cordon — 스케줄링 비활성화

`cordon`은 새로운 Pod이 해당 노드에 스케줄링되는 것만 막아요. 기존 Pod은 그대로 유지돼요.

```bash
# 새 Pod 스케줄링 차단 (기존 Pod 유지)
kubectl cordon node01
```

### kubectl uncordon — 스케줄링 재활성화

유지보수가 끝나면 `uncordon`으로 노드를 다시 사용 가능하게 만들어요.

```bash
# 노드를 다시 스케줄링 가능하게
kubectl uncordon node01
```

> 💡 `uncordon` 후에도 기존에 다른 노드로 이동된 Pod이 자동으로 돌아오지는 않아요. 새로 생성되는 Pod부터 해당 노드에 스케줄링돼요.

### drain vs cordon vs uncordon 비교

| 명령어 | 기존 Pod | 새 Pod 스케줄링 | 용도 |
|--------|---------|---------------|------|
| `kubectl drain` | 다른 노드로 **이동** | **차단** | 노드 유지보수 전 Pod 비우기 |
| `kubectl cordon` | **유지** | **차단** | 노드에 더 이상 Pod 배치 안 함 |
| `kubectl uncordon` | 유지 | **허용** | 유지보수 후 노드 복구 |

### 실전: 노드 OS 업그레이드 순서

```bash
# 1. 노드의 Pod을 안전하게 이동
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data

# 2. 노드에 SSH 접속 후 OS 업그레이드
ssh node01
sudo apt update && sudo apt upgrade -y
sudo reboot

# 3. 노드가 복구된 후 스케줄링 재활성화
kubectl uncordon node01

# 4. 노드 상태 확인
kubectl get nodes
```

---

## 🦥 Kubernetes Software Versions

### K8s 버전 체계

K8s는 **Semantic Versioning** 체계를 따라요.

```
v1.29.3
 │  │  │
 │  │  └── Patch (버그 수정, 보안 패치)
 │  └───── Minor (새 기능 추가, 몇 달마다)
 └──────── Major (큰 변경, 하위 호환 불가능할 수 있음)
```

| 버전 타입 | 예시 | 설명 | 릴리스 주기 |
|----------|------|------|-----------|
| **Alpha** | v1.29.0-alpha.1 | 실험적 기능, 불안정 | - |
| **Beta** | v1.29.0-beta.1 | 테스트 단계, 기본 비활성 | - |
| **Stable** | v1.29.0 | 프로덕션 사용 가능 | ~4개월마다 Minor |

```bash
# 현재 클러스터 버전 확인
kubectl version
kubectl version --short

# 서버 버전만 확인
kubectl get nodes
```

### K8s 컴포넌트 버전 관계

K8s의 핵심 컴포넌트들은 반드시 같은 버전일 필요는 없지만, **호환 규칙**이 있어요.

**기준: kube-apiserver 버전을 X라고 하면**

| 컴포넌트 | 허용 버전 | 예시 (apiserver가 v1.29일 때) |
|----------|----------|-------------------------------|
| **kube-apiserver** | X | v1.29 |
| **controller-manager** | X ~ X-1 | v1.29 또는 v1.28 |
| **kube-scheduler** | X ~ X-1 | v1.29 또는 v1.28 |
| **kubelet** | X ~ X-2 | v1.29, v1.28, 또는 v1.27 |
| **kube-proxy** | X ~ X-2 | v1.29, v1.28, 또는 v1.27 |
| **kubectl** | X+1 ~ X-1 | v1.30, v1.29, 또는 v1.28 |

> 💡 **CKA 포인트**: kube-apiserver가 가장 높은 버전이어야 해요. kubelet은 apiserver보다 최대 2 Minor 버전 낮을 수 있어요. kubectl만 유일하게 apiserver보다 1 버전 높을 수 있어요.

### K8s 지원 정책

K8s는 최근 **3개의 Minor 버전**만 지원해요. 예를 들어 v1.29가 최신이면 v1.27까지 보안 패치를 받을 수 있어요.

---

## 🦥 Cluster Upgrade Process

### 업그레이드 전략

K8s 클러스터 업그레이드는 **한 번에 1 Minor 버전씩**만 올릴 수 있어요.

```
v1.27 → v1.28 → v1.29   ✅ (1 버전씩)
v1.27 → v1.29            ❌ (2 버전 건너뛰기 불가)
```

업그레이드 순서는 항상 이래요.

| 순서 | 대상 | 이유 |
|------|------|------|
| 1 | **Control Plane (Master)** | apiserver가 가장 높은 버전이어야 하니까 |
| 2 | **Worker Nodes** | Master 업그레이드 후 Worker를 올려야 버전 호환 유지 |

### Worker 노드 업그레이드 전략 3가지

| 전략 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **한 번에 전부** | 모든 Worker를 동시에 업그레이드 | 빠름 | **다운타임 발생** |
| **하나씩 순차** | 노드를 하나씩 drain → 업그레이드 → uncordon | 무중단 | 시간이 오래 걸림 |
| **새 노드 교체** | 새 버전 노드 추가 → 기존 노드 제거 | 깔끔, 롤백 쉬움 | 일시적 리소스 2배 |

### kubeadm으로 클러스터 업그레이드 (실습)

#### Step 1: 업그레이드 가능한 버전 확인

```bash
# Control Plane에서 실행
kubeadm upgrade plan
```

#### Step 2: Control Plane 업그레이드

```bash
# 1. kubeadm 업그레이드
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.29.0-1.1
sudo apt-mark hold kubeadm

# 2. 업그레이드 플랜 확인
kubeadm upgrade plan

# 3. 클러스터 업그레이드 적용
sudo kubeadm upgrade apply v1.29.0

# 4. kubelet, kubectl 업그레이드
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
sudo apt-mark hold kubelet kubectl

# 5. kubelet 재시작
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### Step 3: Worker 노드 업그레이드 (노드별 반복)

```bash
# Control Plane에서: Worker 노드 drain
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data

# Worker 노드(node01)에 SSH 접속
ssh node01

# 1. kubeadm 업그레이드
sudo apt-mark unhold kubeadm
sudo apt-get update
sudo apt-get install -y kubeadm=1.29.0-1.1
sudo apt-mark hold kubeadm

# 2. 노드 설정 업그레이드
sudo kubeadm upgrade node

# 3. kubelet, kubectl 업그레이드
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
sudo apt-mark hold kubelet kubectl

# 4. kubelet 재시작
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 5. Control Plane에서: Worker 노드 uncordon
exit   # Worker 노드에서 나가기
kubectl uncordon node01
```

#### Step 4: 업그레이드 확인

```bash
kubectl get nodes
```

```
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   30d   v1.29.0
node01         Ready    <none>          30d   v1.29.0
node02         Ready    <none>          30d   v1.29.0
```

### 업그레이드 핵심 명령어 정리

| 대상 | 핵심 명령어 |
|------|-----------|
| **업그레이드 계획 확인** | `kubeadm upgrade plan` |
| **Control Plane 업그레이드** | `kubeadm upgrade apply v1.29.0` |
| **Worker 노드 업그레이드** | `kubeadm upgrade node` |
| **노드 비우기** | `kubectl drain <node> --ignore-daemonsets` |
| **노드 복구** | `kubectl uncordon <node>` |
| **패키지 버전 고정** | `apt-mark hold kubeadm kubelet kubectl` |

> 💡 **CKA 빈출 패턴**: Control Plane과 Worker의 업그레이드 명령어가 다르다는 걸 기억하세요! Control Plane은 `kubeadm upgrade apply`, Worker는 `kubeadm upgrade node`예요.

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **drain** | 노드의 Pod을 안전하게 이동, `--ignore-daemonsets` 필수 |
| **cordon/uncordon** | 스케줄링 차단/허용, 기존 Pod에 영향 없음 |
| **K8s 버전** | Semantic Versioning, 최근 3개 Minor 버전만 지원 |
| **컴포넌트 호환** | apiserver가 가장 높은 버전, kubelet은 X-2까지 허용 |
| **업그레이드 순서** | Control Plane 먼저 → Worker 순차적으로 |
| **kubeadm 업그레이드** | `upgrade plan` → `upgrade apply` (Master) / `upgrade node` (Worker) |

다음 포스트에서는 **Backup & Restore와 ETCDCTL**을 다뤄볼게요! 🦥
