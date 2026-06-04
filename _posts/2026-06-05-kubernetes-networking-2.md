---
title: "[Kubernetes] K8s Networking (2) — Network Namespace, Docker Networking, CNI"
excerpt: "컨테이너 네트워킹의 핵심 원리! Linux Network Namespace, Docker Bridge, veth pair, CNI 표준까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Networking, Namespace, Docker, CNI, CKA, DevOps]

permalink: /kubernetes/kubernetes-networking-2/

toc: true
toc_sticky: true

date: 2026-06-05
last_modified_at: 2026-06-05
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Networking 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [Networking (1)](/kubernetes/kubernetes-networking-1/)에서 네트워크 기초를 다뤘어요!

---

## 🦥 Network Namespace

### Network Namespace란?

Linux Network Namespace는 **네트워크 환경을 격리**하는 기술이에요. 각 Namespace는 독립된 인터페이스, 라우팅 테이블, ARP 테이블을 가져요.

컨테이너가 바로 이 Network Namespace를 사용해서 네트워크를 격리해요.

| 구분 | Host | Network Namespace |
|------|------|-------------------|
| 인터페이스 | eth0, lo 등 전체 | 자체 가상 인터페이스 |
| 라우팅 테이블 | 호스트 전체 라우트 | 독립된 라우트 |
| ARP 테이블 | 호스트 전체 ARP | 독립된 ARP |
| 프로세스 관점 | 모든 네트워크 보임 | 자기 Namespace만 보임 |

### Namespace 생성 및 관리

```bash
# Network Namespace 생성
ip netns add red
ip netns add blue

# Namespace 목록 확인
ip netns

# Namespace 내에서 명령어 실행
ip netns exec red ip link
ip netns exec red ip route
ip netns exec red arp
```

### 두 Namespace 연결 — veth pair

두 Namespace를 직접 연결하려면 **veth (Virtual Ethernet) pair**를 사용해요. 가상 케이블이라고 생각하면 돼요.

```bash
# veth pair 생성 (가상 케이블의 양쪽 끝)
ip link add veth-red type veth peer name veth-blue

# 각 끝을 해당 Namespace에 연결
ip link set veth-red netns red
ip link set veth-blue netns blue

# 각 Namespace에서 IP 할당
ip netns exec red ip addr add 192.168.15.1/24 dev veth-red
ip netns exec blue ip addr add 192.168.15.2/24 dev veth-blue

# 인터페이스 활성화
ip netns exec red ip link set veth-red up
ip netns exec blue ip link set veth-blue up

# 통신 확인
ip netns exec red ping 192.168.15.2
```

### 여러 Namespace 연결 — Virtual Bridge

Namespace가 많아지면 veth pair로 일일이 연결할 수 없어요. **Virtual Bridge(가상 스위치)**를 사용해요.

```bash
# Linux Bridge 생성
ip link add v-net-0 type bridge
ip link set dev v-net-0 up

# veth pair 생성 (Namespace ↔ Bridge)
ip link add veth-red type veth peer name veth-red-br
ip link add veth-blue type veth peer name veth-blue-br

# Namespace 쪽 연결
ip link set veth-red netns red
ip link set veth-blue netns blue

# Bridge 쪽 연결
ip link set veth-red-br master v-net-0
ip link set veth-blue-br master v-net-0

# IP 할당 및 활성화
ip netns exec red ip addr add 192.168.15.1/24 dev veth-red
ip netns exec blue ip addr add 192.168.15.2/24 dev veth-blue
ip netns exec red ip link set veth-red up
ip netns exec blue ip link set veth-blue up
```

### Bridge에 IP 할당 — Host와 통신

Bridge 자체에 IP를 할당하면 Host에서도 Namespace로 접근할 수 있어요.

```bash
# Bridge에 IP 할당
ip addr add 192.168.15.5/24 dev v-net-0
```

### 외부 네트워크 연결

Namespace에서 외부 네트워크로 나가려면 **NAT(iptables)**와 라우팅이 필요해요.

```bash
# Namespace에 Default Gateway 설정
ip netns exec red ip route add default via 192.168.15.5

# Host에서 NAT 설정 (MASQUERADE)
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
```

> 💡 이 모든 과정이 Docker와 K8s가 컨테이너/Pod 네트워킹을 구현하는 기본 원리예요!

---

## 🦥 Docker Networking

Docker는 위에서 배운 Network Namespace와 Bridge를 사용해서 컨테이너 네트워크를 구현해요.

### Docker Network 모드

| 모드 | 설명 | 사용 사례 |
|------|------|----------|
| `none` | 네트워크 완전 격리, 외부 통신 불가 | 보안이 중요한 작업 |
| `host` | 호스트 네트워크 공유, 포트 격리 없음 | 최대 성능 필요 시 |
| `bridge` | 가상 브릿지 네트워크 (기본값) | 일반적인 컨테이너 |

```bash
# none 모드
docker run --network none nginx

# host 모드
docker run --network host nginx

# bridge 모드 (기본)
docker run nginx
```

### Bridge 네트워크 동작 원리

Docker는 설치 시 `docker0`라는 Bridge를 자동으로 생성해요.

```bash
# docker0 Bridge 확인
ip link show docker0
ip addr show docker0    # 보통 172.17.0.1/16
```

컨테이너가 생성될 때 Docker가 하는 일이에요.

| 단계 | 동작 |
|------|------|
| 1 | 컨테이너용 Network Namespace 생성 |
| 2 | veth pair 생성 |
| 3 | 한쪽은 컨테이너 Namespace에 연결 (eth0) |
| 4 | 다른 쪽은 docker0 Bridge에 연결 |
| 5 | 컨테이너에 IP 할당 (172.17.0.x) |

### Port Mapping

컨테이너 내부 포트를 외부에서 접근하려면 **Port Mapping**이 필요해요.

```bash
# 호스트 8080 → 컨테이너 80
docker run -p 8080:80 nginx
```

Docker는 내부적으로 **iptables NAT 규칙**을 생성해서 포트를 매핑해요.

```bash
# Docker가 생성한 iptables 규칙 확인
iptables -t nat -L -n | grep 8080

# 결과 예시
DNAT tcp -- 0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80
```

---

## 🦥 CNI (Container Network Interface)

### CNI가 필요한 이유

Docker, rkt, K8s 등 모든 Container Runtime이 비슷한 네트워크 설정을 해요. 매번 같은 코드를 구현하는 건 비효율적이에요.

| 공통 작업 | 설명 |
|----------|------|
| Network Namespace 생성 | 컨테이너 격리 |
| Bridge 생성/관리 | 컨테이너 간 통신 |
| veth pair 연결 | Namespace ↔ Bridge |
| IP 할당 | IPAM (IP Address Management) |
| 라우팅 설정 | 외부 네트워크 연결 |

### CNI란?

CNI는 **Container Runtime과 네트워크 플러그인 사이의 표준 인터페이스**예요. Container Runtime은 CNI 표준을 따르는 플러그인을 호출하기만 하면 돼요.

| CNI 표준 정의 | 내용 |
|-------------|------|
| Container Runtime 책임 | Namespace 생성, CNI 플러그인 호출 |
| Plugin 책임 | 네트워크 설정 (Bridge, IP 할당, 라우팅) |
| ADD 동작 | 컨테이너가 네트워크에 참여할 때 |
| DEL 동작 | 컨테이너가 네트워크에서 떠날 때 |

### CNI 플러그인 종류

| 플러그인 | 특징 |
|---------|------|
| **bridge** | 기본 Linux Bridge 기반 |
| **vlan** | VLAN 태깅 |
| **ipvlan** | IP 기반 가상 인터페이스 |
| **macvlan** | MAC 기반 가상 인터페이스 |
| **Weave** | Overlay 네트워크, 분산 라우팅 |
| **Flannel** | 간단한 Overlay 네트워크 |
| **Calico** | BGP 기반, NetworkPolicy 지원 |
| **Cilium** | eBPF 기반, 고성능 |

### Docker와 CNI

흥미롭게도, **Docker는 CNI를 직접 구현하지 않아요**. Docker는 자체 네트워크 표준인 **CNM (Container Network Model)**을 사용해요.

| 런타임 | 네트워크 표준 |
|--------|-------------|
| Docker | CNM (자체 표준) |
| K8s (containerd, CRI-O) | CNI |

그래서 K8s가 Docker를 Container Runtime으로 사용할 때는 `docker run`의 네트워크 모드를 `none`으로 설정하고, 별도로 CNI 플러그인을 호출해서 네트워크를 설정했어요.

```bash
# K8s가 Docker를 사용할 때의 흐름 (과거)
docker run --network=none nginx
bridge add <container-id> /var/run/netns/<container-id>
```

> 💡 현재 K8s는 Docker를 직접 지원하지 않고 **containerd**나 **CRI-O**를 사용해요. 이들은 CNI를 네이티브로 지원해요.

---

## 🦥 CNI 설정 파일

CNI 플러그인의 설정은 JSON 파일로 관리해요.

```json
{
  "cniVersion": "0.3.1",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.22.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

| 필드 | 설명 |
|------|------|
| `type` | 사용할 CNI 플러그인 |
| `bridge` | Bridge 이름 |
| `isGateway` | Bridge에 IP 할당 (Gateway 역할) |
| `ipMasq` | NAT 설정 여부 |
| `ipam` | IP 할당 방식 |
| `ipam.type` | IPAM 플러그인 (host-local, dhcp 등) |
| `ipam.subnet` | 할당할 IP 대역 |

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Network Namespace** | 네트워크 환경 격리 (인터페이스, 라우팅, ARP 독립) |
| **veth pair** | 두 Namespace를 연결하는 가상 케이블 |
| **Virtual Bridge** | 여러 Namespace를 연결하는 가상 스위치 |
| **Docker Networking** | docker0 Bridge + veth pair + iptables NAT |
| **Docker Network 모드** | none (격리), host (공유), bridge (기본) |
| **Port Mapping** | iptables DNAT 규칙으로 구현 |
| **CNI** | Container Runtime ↔ Network Plugin 표준 인터페이스 |
| **CNI Plugin** | Weave, Flannel, Calico, Cilium 등 |
| **Docker vs CNI** | Docker는 CNM 사용, K8s는 CNI 사용 |

다음 포스트에서는 **Cluster Networking, Pod Networking, CNI in K8s**를 다룰게요! 🦥
