---
title: "[Kubernetes] K8s Networking (1) — 네트워크 기초: Switching, Routing, Gateway, DNS"
excerpt: "K8s 네트워킹의 토대! Linux 네트워크 기초부터 Switching, Routing, Gateway, DNS, CoreDNS까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Networking, DNS, CoreDNS, Routing, CKA, DevOps]

permalink: /kubernetes/kubernetes-networking-1/

toc: true
toc_sticky: true

date: 2026-06-04
last_modified_at: 2026-06-04
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Networking 섹션을 기반으로 정리한 내용이에요.

---

## 🦥 왜 네트워크 기초가 필요할까?

K8s 네트워킹을 이해하려면 Linux 네트워크 기초 지식이 필수예요. Pod 간 통신, Service 라우팅, DNS 해석 — 모두 이 기초 위에 동작해요.

---

## 🦥 Switching (스위칭)

### 같은 네트워크 내 통신

두 컴퓨터가 같은 네트워크에 있을 때, **Switch**를 통해 통신해요.

```bash
# 호스트의 네트워크 인터페이스 확인
ip link

# 인터페이스에 IP 주소 할당
ip addr add 192.168.1.10/24 dev eth0
ip addr add 192.168.1.11/24 dev eth0
```

| 명령어 | 설명 |
|--------|------|
| `ip link` | 네트워크 인터페이스 목록 확인 |
| `ip addr` | 인터페이스의 IP 주소 확인 |
| `ip addr add` | IP 주소 할당 |

> 💡 Switch는 같은 네트워크(같은 서브넷) 내의 호스트끼리만 통신할 수 있어요.

---

## 🦥 Routing (라우팅)

### 다른 네트워크 간 통신

서로 다른 네트워크(예: 192.168.1.0/24와 192.168.2.0/24)를 연결하려면 **Router**가 필요해요. Router는 두 네트워크에 각각 IP를 가지고 있어요.

```
네트워크 A (192.168.1.0/24)
  ├── Host A: 192.168.1.10
  └── Router: 192.168.1.1
          │
네트워크 B (192.168.2.0/24)
  ├── Router: 192.168.2.1
  └── Host B: 192.168.2.10
```

### Routing Table (라우팅 테이블)

각 호스트는 **라우팅 테이블**을 통해 어떤 네트워크로 가려면 어디로 보내야 하는지 알아요.

```bash
# 라우팅 테이블 확인
ip route
# 또는
route

# 라우트 추가 — "192.168.2.0 네트워크로 가려면 192.168.1.1(Router)로 보내라"
ip route add 192.168.2.0/24 via 192.168.1.1
```

| 명령어 | 설명 |
|--------|------|
| `ip route` | 라우팅 테이블 확인 |
| `ip route add <network> via <gateway>` | 라우트 추가 |
| `route` | 라우팅 테이블 확인 (구버전) |

### Default Gateway

모든 목적지 네트워크에 대해 일일이 라우트를 추가하는 건 비현실적이에요. **Default Gateway**를 설정하면 알 수 없는 목적지는 모두 이 게이트웨이로 보내요.

```bash
# Default Gateway 설정
ip route add default via 192.168.1.1

# 동일한 의미
ip route add 0.0.0.0/0 via 192.168.1.1
```

> 💡 `0.0.0.0/0`은 "모든 네트워크"를 의미해요. 라우팅 테이블에서 매칭되는 규칙이 없으면 default gateway로 보내요.

### IP Forwarding

Linux 호스트를 라우터처럼 사용하려면 **IP Forwarding**을 활성화해야 해요. 기본적으로 비활성화되어 있어요.

```bash
# 현재 설정 확인 (0=비활성, 1=활성)
cat /proc/sys/net/ipv4/ip_forward

# 임시 활성화
echo 1 > /proc/sys/net/ipv4/ip_forward

# 영구 활성화
# /etc/sysctl.conf에 추가:
net.ipv4.ip_forward = 1

# 적용
sysctl -p
```

> 💡 **CKA 포인트**: K8s 노드는 Pod 간 트래픽을 전달해야 하므로 IP Forwarding이 활성화되어 있어야 해요.

---

## 🦥 DNS (Domain Name System)

### /etc/hosts — 로컬 이름 해석

가장 간단한 이름 해석 방법은 `/etc/hosts` 파일이에요.

```bash
# /etc/hosts
192.168.1.10  web-server
192.168.1.11  db-server
192.168.1.12  nfs-server
```

```bash
# 이제 IP 대신 이름으로 접근 가능
ping web-server
ssh db-server
```

| 장점 | 한계 |
|------|------|
| 설정이 간단함 | 호스트가 많아지면 관리 불가 |
| 외부 의존 없음 | 모든 호스트에 동일 설정 필요 |

### DNS Server — 중앙 집중 이름 해석

호스트가 많아지면 **DNS Server**를 사용해서 이름 해석을 중앙에서 관리해요.

```bash
# /etc/resolv.conf — DNS Server 지정
nameserver 192.168.1.100
nameserver 8.8.8.8          # 보조 DNS (Google)
```

### 이름 해석 순서

```bash
# /etc/nsswitch.conf
hosts: files dns
```

| 순서 | 소스 | 설명 |
|------|------|------|
| 1 | `files` | `/etc/hosts` 파일 먼저 확인 |
| 2 | `dns` | DNS Server에 질의 |

> 💡 `/etc/hosts`에 있는 항목이 DNS Server보다 우선해요. 이 순서는 `/etc/nsswitch.conf`에서 변경할 수 있어요.

### Domain Name 구조

```
www.google.com
 │    │     │
 │    │     └── Top-Level Domain (TLD): .com
 │    └── Domain: google
 └── Subdomain: www
```

| TLD | 용도 |
|-----|------|
| `.com` | 상업용 |
| `.org` | 비영리 조직 |
| `.net` | 네트워크 |
| `.io` | 기술/스타트업 |
| `.edu` | 교육기관 |

### Search Domain

```bash
# /etc/resolv.conf
nameserver 192.168.1.100
search mycompany.com prod.mycompany.com
```

`search` 설정이 있으면 짧은 이름으로도 접근할 수 있어요.

| 입력 | 시도하는 DNS 질의 |
|------|-----------------|
| `web-server` | web-server → web-server.mycompany.com → web-server.prod.mycompany.com |
| `web-server.mycompany.com` | 정확한 FQDN으로 바로 질의 |

### DNS Record Types

| Record Type | 설명 | 예시 |
|------------|------|------|
| `A` | 이름 → IPv4 주소 | web.example.com → 192.168.1.10 |
| `AAAA` | 이름 → IPv6 주소 | web.example.com → 2001:db8::1 |
| `CNAME` | 이름 → 다른 이름 (별칭) | food.web.com → eat.web.com |

### DNS 관련 도구

```bash
# DNS 질의
nslookup web-server.example.com

# 상세 DNS 질의
dig web-server.example.com
```

| 도구 | 특징 |
|------|------|
| `nslookup` | DNS Server만 질의 (`/etc/hosts` 무시) |
| `dig` | 상세한 DNS 정보 제공 (TTL, Authority 등) |

---

## 🦥 CoreDNS

### CoreDNS란?

CoreDNS는 Go로 작성된 **유연하고 확장 가능한 DNS Server**예요. K8s의 기본 DNS Server로 사용돼요.

### CoreDNS 기본 설정

```bash
# CoreDNS 설치 및 실행
./coredns
```

CoreDNS의 설정 파일은 **Corefile**이에요.

```
# Corefile
.:53 {
    hosts /etc/hosts     # /etc/hosts 파일을 DNS 레코드로 서빙
    forward . 8.8.8.8    # 해석 못 하는 건 8.8.8.8로 포워딩
    log                  # 질의 로깅
    errors               # 에러 로깅
}
```

| 설정 | 설명 |
|------|------|
| `.:53` | 모든 도메인에 대해 53번 포트에서 리슨 |
| `hosts` | hosts 파일 기반 레코드 |
| `forward` | 해석 못 하는 질의를 다른 DNS로 포워딩 |
| `log` | 질의 로그 기록 |
| `cache` | DNS 응답 캐싱 |

> 💡 CoreDNS가 K8s에서 어떻게 사용되는지는 [Networking (4)](/kubernetes/kubernetes-networking-4/)에서 자세히 다룰게요.

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Switching** | 같은 네트워크 내 통신, `ip link`, `ip addr` |
| **Routing** | 다른 네트워크 간 통신, `ip route add` |
| **Default Gateway** | 알 수 없는 목적지의 기본 경로 (`0.0.0.0/0`) |
| **IP Forwarding** | Linux를 라우터로 사용하려면 활성화 필수 |
| **DNS** | `/etc/hosts` (로컬) → `/etc/resolv.conf` (DNS Server) |
| **이름 해석 순서** | `/etc/nsswitch.conf`에서 설정 (기본: files → dns) |
| **CoreDNS** | Go 기반 DNS Server, Corefile로 설정, K8s 기본 DNS |

다음 포스트에서는 **Network Namespace, Docker Networking, CNI**를 다룰게요! 🦥
