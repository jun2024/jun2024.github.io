---
title: "[Kubernetes] K8s Networking (4) — Service Networking, DNS & CoreDNS in K8s"
excerpt: "K8s Service의 네트워크 원리! ClusterIP/NodePort가 동작하는 방식, kube-proxy, K8s DNS 체계와 CoreDNS까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Networking, Service, DNS, CoreDNS, kube-proxy, CKA, DevOps]

permalink: /kubernetes/kubernetes-networking-4/

toc: true
toc_sticky: true

date: 2026-06-07
last_modified_at: 2026-06-07
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Networking 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [Networking (3)](/kubernetes/kubernetes-networking-3/)에서 Cluster Networking, Pod Networking, CNI를 다뤘어요!

---

## 🦥 Service Networking

### Service는 어떻게 동작할까?

Pod은 특정 노드에 존재하지만, **Service는 클러스터 전체에 존재하는 가상 객체**예요. 어떤 노드에서든 Service IP로 접근할 수 있어요.

| 구분 | Pod | Service |
|------|-----|---------|
| 존재 위치 | 특정 노드 | 클러스터 전체 (가상) |
| IP 할당 | CNI 플러그인 | kube-apiserver |
| 네트워크 구현 | Bridge, veth pair | iptables/IPVS 규칙 |
| 담당 컴포넌트 | kubelet + CNI | kube-proxy |

### kube-proxy의 역할

Service를 생성하면 kube-proxy가 각 노드에 **네트워크 규칙**을 만들어요. 이 규칙이 Service IP로 들어오는 트래픽을 실제 Pod IP로 전달해요.

| kube-proxy 모드 | 설명 | 특징 |
|----------------|------|------|
| `userspace` | kube-proxy가 직접 프록시 | 느림, 구버전 |
| `iptables` | iptables 규칙 생성 (기본값) | 성능 좋음, 대부분 사용 |
| `ipvs` | IPVS 규칙 생성 | 대규모 클러스터에 적합, 다양한 LB 알고리즘 |

```bash
# kube-proxy 모드 확인
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# kube-proxy 로그에서 모드 확인
kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using"
```

### Service가 생성되는 과정

| 단계 | 동작 | 담당 |
|------|------|------|
| 1 | Service 생성 요청 | 사용자 (kubectl) |
| 2 | Service 객체 생성, ClusterIP 할당 | kube-apiserver |
| 3 | Endpoints 객체 생성 (Pod IP 매핑) | Endpoint Controller |
| 4 | 모든 노드에 iptables 규칙 생성 | kube-proxy |
| 5 | Service IP로 트래픽 → Pod IP로 전달 | iptables (커널) |

### ClusterIP Service의 iptables 규칙

```bash
# Service 생성
kubectl create service clusterip my-svc --tcp=80:80

# iptables 규칙 확인
iptables -t nat -L -n | grep my-svc
```

```
# iptables 규칙 예시
Chain KUBE-SERVICES
  -d 10.96.0.100/32 -p tcp --dport 80 -j KUBE-SVC-XXXX

Chain KUBE-SVC-XXXX    # 로드밸런싱 (확률 기반)
  -m statistic --mode random --probability 0.333 -j KUBE-SEP-AAA
  -m statistic --mode random --probability 0.500 -j KUBE-SEP-BBB
  -j KUBE-SEP-CCC

Chain KUBE-SEP-AAA     # 실제 Pod으로 DNAT
  -p tcp -j DNAT --to-destination 10.244.1.2:80
```

> 💡 `iptables` 모드에서 로드밸런싱은 **확률 기반(random)**이에요. 3개 Pod이 있으면 각각 33%, 50%, 100% 확률로 선택돼요.

### NodePort Service

```bash
# NodePort iptables 규칙 확인
iptables -t nat -L -n | grep NodePort
```

NodePort는 ClusterIP 규칙에 추가로 **노드 포트(30000-32767)에 대한 규칙**을 만들어요.

### Service IP 대역

```bash
# Service IP 대역 확인 (kube-apiserver 설정)
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range

# 기본값: --service-cluster-ip-range=10.96.0.0/12
```

| 대역 | 용도 | 설정 위치 |
|------|------|----------|
| Service IP (예: 10.96.0.0/12) | ClusterIP 할당 | kube-apiserver `--service-cluster-ip-range` |
| Pod IP (예: 10.244.0.0/16) | Pod IP 할당 | CNI 플러그인 설정 |

> ⚠️ **주의**: Service IP 대역과 Pod IP 대역이 겹치면 안 돼요!

---

## 🦥 DNS in Kubernetes

### K8s 내장 DNS

K8s 클러스터에는 **내장 DNS Server**가 있어요. Pod과 Service에 자동으로 DNS 레코드를 생성해줘요.

### Service DNS 레코드

Service를 생성하면 자동으로 DNS 레코드가 만들어져요.

| DNS 형식 | 예시 |
|----------|------|
| `<service-name>` | `my-svc` (같은 Namespace) |
| `<service-name>.<namespace>` | `my-svc.default` |
| `<service-name>.<namespace>.svc` | `my-svc.default.svc` |
| `<service-name>.<namespace>.svc.cluster.local` | `my-svc.default.svc.cluster.local` (FQDN) |

```bash
# 같은 Namespace에서 접근
curl http://my-svc

# 다른 Namespace에서 접근
curl http://my-svc.other-namespace

# FQDN으로 접근
curl http://my-svc.default.svc.cluster.local
```

### Pod DNS 레코드

Pod에도 DNS 레코드가 만들어지지만, IP의 점(.)을 하이픈(-)으로 바꾼 형식이에요.

| 리소스 | DNS 형식 | 예시 |
|--------|----------|------|
| Service | `<name>.<ns>.svc.cluster.local` | `my-svc.default.svc.cluster.local` |
| Pod | `<ip-with-dashes>.<ns>.pod.cluster.local` | `10-244-1-2.default.pod.cluster.local` |

> 💡 Pod DNS는 잘 사용하지 않아요. 보통 Service를 통해 접근하기 때문이에요.

---

## 🦥 CoreDNS in Kubernetes

### K8s의 DNS Server: CoreDNS

K8s 1.13부터 **CoreDNS**가 기본 DNS Server예요. `kube-system` Namespace에 Deployment로 배포돼요.

```bash
# CoreDNS Pod 확인
kubectl get pods -n kube-system | grep coredns

# CoreDNS Deployment 확인
kubectl get deployment coredns -n kube-system
```

### CoreDNS 설정 — Corefile

CoreDNS의 설정은 **ConfigMap**으로 관리돼요.

```bash
# Corefile ConfigMap 확인
kubectl get configmap coredns -n kube-system -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

| 플러그인 | 역할 |
|---------|------|
| `kubernetes` | K8s Service/Pod DNS 레코드 제공 |
| `forward` | 클러스터 외부 도메인은 업스트림 DNS로 포워딩 |
| `cache` | DNS 응답 캐싱 (30초) |
| `health` | Health check 엔드포인트 (/health) |
| `ready` | Readiness check (/ready) |
| `prometheus` | 메트릭 노출 (9153 포트) |
| `loop` | 무한 루프 감지 |
| `reload` | Corefile 변경 시 자동 리로드 |

### CoreDNS Service

CoreDNS는 **kube-dns**라는 이름의 Service로 노출돼요.

```bash
# CoreDNS Service 확인
kubectl get svc -n kube-system
# NAME       TYPE        CLUSTER-IP   PORT(S)
# kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP,9153/TCP
```

### Pod의 DNS 설정

Pod이 생성되면 kubelet이 자동으로 `/etc/resolv.conf`에 CoreDNS를 nameserver로 설정해요.

```bash
# Pod 내부에서 확인
kubectl exec my-pod -- cat /etc/resolv.conf
```

```
nameserver 10.96.0.10          # CoreDNS Service IP
search default.svc.cluster.local svc.cluster.local cluster.local
```

| 항목 | 설명 |
|------|------|
| `nameserver` | CoreDNS Service의 ClusterIP |
| `search` | 짧은 이름에 자동으로 붙는 도메인 접미사 |

`search` 설정 덕분에 짧은 이름으로도 Service에 접근할 수 있어요.

| 입력 | DNS 질의 순서 |
|------|-------------|
| `my-svc` | my-svc.default.svc.cluster.local → my-svc.svc.cluster.local → ... |
| `my-svc.other-ns` | my-svc.other-ns.svc.cluster.local → ... |

### CoreDNS 문제 해결

```bash
# CoreDNS Pod 상태 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS 로그 확인
kubectl logs -n kube-system -l k8s-app=kube-dns

# DNS 해석 테스트
kubectl run test-dns --image=busybox:1.28 --rm -it -- nslookup my-svc
kubectl run test-dns --image=busybox:1.28 --rm -it -- nslookup kubernetes.default
```

---

## 🦥 CKA 시험 대비 — Service & DNS 치트시트

```bash
# Service IP 대역 확인
grep service-cluster-ip-range /etc/kubernetes/manifests/kube-apiserver.yaml

# Pod IP 대역 확인
kubectl cluster-info dump | grep -m 1 cluster-cidr

# kube-proxy 모드 확인
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# iptables 규칙에서 Service 찾기
iptables -t nat -L -n | grep <service-name>

# CoreDNS 설정 확인
kubectl get configmap coredns -n kube-system -o yaml

# DNS 테스트
kubectl run test --image=busybox:1.28 --rm -it -- nslookup <service-name>
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Service** | 클러스터 전체에 존재하는 가상 객체, kube-proxy가 구현 |
| **kube-proxy 모드** | userspace, iptables (기본), ipvs |
| **iptables 모드** | 확률 기반 로드밸런싱, DNAT으로 Pod IP 전달 |
| **Service DNS** | `<name>.<ns>.svc.cluster.local` |
| **Pod DNS** | `<ip-dashes>.<ns>.pod.cluster.local` (거의 안 씀) |
| **CoreDNS** | K8s 기본 DNS, ConfigMap(Corefile)로 설정 |
| **kube-dns Service** | CoreDNS의 ClusterIP, Pod의 nameserver |
| **search 도메인** | 짧은 이름으로 Service 접근 가능 |

다음 포스트에서는 **Ingress**를 다룰게요! 🦥
