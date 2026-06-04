---
title: "[Kubernetes] K8s Networking (3) — Cluster Networking, Pod Networking, CNI in K8s"
excerpt: "K8s 클러스터의 네트워크 구조! 노드 간 통신, Pod 네트워킹 모델, CNI 플러그인(Weave) 동작 원리까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Networking, Pod, CNI, Weave, IPAM, CKA, DevOps]

permalink: /kubernetes/kubernetes-networking-3/

toc: true
toc_sticky: true

date: 2026-06-06
last_modified_at: 2026-06-06
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Networking 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [Networking (2)](/kubernetes/kubernetes-networking-2/)에서 Network Namespace, Docker Networking, CNI를 다뤘어요!

---

## 🦥 Cluster Networking

### K8s 클러스터의 네트워크 요구사항

K8s 클러스터를 구성하려면 각 노드에 특정 포트들이 열려 있어야 해요.

### Control Plane (Master) 노드 포트

| 포트 | 컴포넌트 | 프로토콜 | 용도 |
|------|---------|---------|------|
| 6443 | kube-apiserver | TCP | K8s API 접근 |
| 2379-2380 | etcd | TCP | etcd 클라이언트/피어 통신 |
| 10250 | kubelet | TCP | kubelet API |
| 10259 | kube-scheduler | TCP | 스케줄러 |
| 10257 | kube-controller-manager | TCP | 컨트롤러 매니저 |

### Worker 노드 포트

| 포트 | 컴포넌트 | 프로토콜 | 용도 |
|------|---------|---------|------|
| 10250 | kubelet | TCP | kubelet API |
| 10256 | kube-proxy | TCP | kube-proxy health check |
| 30000-32767 | NodePort Service | TCP | 외부 서비스 접근 |

### 노드 네트워크 확인

```bash
# 노드의 IP 주소 확인
ip addr show eth0
kubectl get nodes -o wide

# 열린 포트 확인
netstat -tlnp | grep 6443
ss -tlnp | grep 6443
```

> 💡 **CKA 포인트**: 6443(apiserver), 2379(etcd), 10250(kubelet), 30000-32767(NodePort) — 이 포트 번호는 반드시 기억하세요!

---

## 🦥 Pod Networking

### K8s의 네트워킹 모델

K8s는 Pod 네트워킹에 대해 3가지 요구사항을 정해요. "어떻게"는 정하지 않고 "무엇"만 정해요.

| 요구사항 | 설명 |
|---------|------|
| **Pod 간 통신** | 모든 Pod은 NAT 없이 다른 모든 Pod과 통신 가능 |
| **Node-Pod 통신** | 모든 Node는 NAT 없이 모든 Pod과 통신 가능 |
| **Pod 자기 인식** | Pod이 보는 자신의 IP = 다른 Pod이 보는 그 Pod의 IP |

> 💡 K8s는 이 요구사항을 직접 구현하지 않아요. **CNI 플러그인**이 구현해요.

### Pod Networking 동작 원리

각 노드에서 일어나는 일이에요.

| 단계 | 동작 |
|------|------|
| 1 | 노드에 Bridge 네트워크 생성 (예: `cni0`) |
| 2 | Pod 생성 시 Network Namespace 생성 |
| 3 | veth pair로 Pod Namespace ↔ Bridge 연결 |
| 4 | Pod에 IP 할당 |
| 5 | 라우팅 규칙 설정 |

### 같은 노드 내 Pod 간 통신

같은 노드의 Pod들은 Bridge를 통해 직접 통신해요.

```
Pod A (10.244.1.2) ─── veth ─── cni0 Bridge ─── veth ─── Pod B (10.244.1.3)
```

### 다른 노드의 Pod 간 통신

다른 노드의 Pod과 통신하려면 **노드 간 라우팅**이 필요해요.

| 방법 | 설명 |
|------|------|
| 수동 라우팅 | 각 노드에 다른 노드의 Pod 대역 라우트 추가 |
| Overlay 네트워크 | 패킷을 캡슐화하여 노드 간 전달 (Weave, Flannel 등) |
| BGP | 라우팅 프로토콜로 Pod 대역 광고 (Calico) |

```bash
# 수동 라우팅 예시 (비추천, 노드가 많으면 불가)
# Node1에서: Node2의 Pod 대역은 Node2로 보내라
ip route add 10.244.2.0/24 via 192.168.1.12

# Node2에서: Node1의 Pod 대역은 Node1로 보내라
ip route add 10.244.1.0/24 via 192.168.1.11
```

---

## 🦥 CNI in Kubernetes

### K8s에서 CNI 설정

kubelet이 Pod을 생성할 때 CNI 플러그인을 호출해요. kubelet의 CNI 관련 설정을 확인해볼게요.

```bash
# kubelet의 CNI 설정 확인
ps aux | grep kubelet | grep cni

# 주요 설정값
# --cni-bin-dir=/opt/cni/bin       (CNI 플러그인 바이너리 경로)
# --cni-conf-dir=/etc/cni/net.d    (CNI 설정 파일 경로)
# --network-plugin=cni             (CNI 사용)
```

| 경로 | 설명 |
|------|------|
| `/opt/cni/bin/` | CNI 플러그인 바이너리 (bridge, flannel, weave 등) |
| `/etc/cni/net.d/` | CNI 설정 파일 (어떤 플러그인 사용할지) |

```bash
# 설치된 CNI 플러그인 확인
ls /opt/cni/bin/

# 현재 사용 중인 CNI 설정 확인
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-weave.conflist
```

### CNI 플러그인 호출 흐름

| 단계 | 동작 | 담당 |
|------|------|------|
| 1 | Pod 생성 요청 | kube-apiserver |
| 2 | Pod을 노드에 스케줄 | kube-scheduler |
| 3 | kubelet이 컨테이너 생성 | kubelet → Container Runtime |
| 4 | Container Runtime이 CNI 호출 | Container Runtime → CNI Plugin |
| 5 | 네트워크 설정 완료 | CNI Plugin |

---

## 🦥 CNI Weave

### Weave 동작 원리

Weave는 각 노드에 **Agent**를 배포하고, Agent들이 서로 통신하면서 **Overlay 네트워크**를 구성해요.

| 구성 요소 | 역할 |
|----------|------|
| Weave Agent (각 노드) | 패킷 가로채기, 캡슐화/디캡슐화 |
| Weave Bridge (`weave`) | 노드 내 Pod 연결 |
| Overlay Network | 노드 간 터널링 |

### Weave의 패킷 전달 과정

| 단계 | 동작 |
|------|------|
| 1 | Pod A가 Pod B(다른 노드)로 패킷 전송 |
| 2 | Weave Agent가 패킷 가로채기 |
| 3 | 목적지 Pod이 어느 노드에 있는지 확인 |
| 4 | 원래 패킷을 새 패킷으로 캡슐화 (Encapsulation) |
| 5 | 목적지 노드로 전송 |
| 6 | 상대 노드의 Weave Agent가 디캡슐화 |
| 7 | Pod B에 원래 패킷 전달 |

### Weave 배포

```bash
# Weave CNI 설치 (DaemonSet으로 배포)
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Weave Pod 확인
kubectl get pods -n kube-system | grep weave

# Weave 로그 확인
kubectl logs -n kube-system weave-net-xxxxx -c weave
```

Weave는 **DaemonSet**으로 배포되어 모든 노드에 Agent Pod이 실행돼요.

---

## 🦥 IPAM (IP Address Management)

### IP 할당은 누가 할까?

CNI 플러그인이 Pod에 IP를 할당해요. IP가 충돌하지 않도록 관리하는 게 **IPAM**이에요.

| IPAM 방식 | 설명 |
|----------|------|
| `host-local` | 각 노드에 서브넷을 할당하고, 노드가 로컬에서 IP 관리 |
| `dhcp` | DHCP 서버에서 IP 할당 |

### Weave의 IPAM

Weave는 기본적으로 `10.32.0.0/12` 대역을 사용해요. 이 대역은 10.32.0.1 ~ 10.47.255.254 범위예요.

| 항목 | 값 |
|------|-----|
| 기본 대역 | 10.32.0.0/12 |
| IP 개수 | 약 100만 개 |
| 분배 방식 | 각 노드에 대역을 나누어 할당 |

```bash
# Weave의 IP 할당 대역 확인
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status

# Pod의 IP 확인
kubectl get pods -o wide
```

### CKA 시험에서 자주 묻는 것

```bash
# 어떤 CNI 플러그인이 사용되고 있는지 확인
ls /etc/cni/net.d/

# CNI 설정 파일 내용 확인
cat /etc/cni/net.d/10-weave.conflist

# Pod에 할당된 IP 대역 확인
kubectl get pods -A -o wide | awk '{print $7}' | sort -u

# 노드의 내부 IP 확인
kubectl get nodes -o wide
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Cluster Networking** | Master: 6443, 2379, 10250 / Worker: 10250, 30000-32767 |
| **Pod 네트워킹 모델** | 모든 Pod은 NAT 없이 서로 통신 가능해야 함 |
| **같은 노드 Pod** | Bridge(cni0)를 통해 직접 통신 |
| **다른 노드 Pod** | CNI 플러그인이 Overlay/라우팅으로 연결 |
| **CNI 설정** | `/opt/cni/bin/` (바이너리), `/etc/cni/net.d/` (설정) |
| **Weave** | DaemonSet 배포, Overlay 네트워크, 패킷 캡슐화 |
| **IPAM** | Pod IP 할당 관리, Weave 기본 10.32.0.0/12 |

다음 포스트에서는 **Service Networking, K8s DNS & CoreDNS**를 다룰게요! 🦥
