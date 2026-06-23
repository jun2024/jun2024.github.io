---
title: "[Kubernetes] kubeadm으로 K8s 클러스터 설치하기 — 완벽 실습 가이드"
excerpt: "CKA 시험 필수! kubeadm으로 K8s 클러스터를 처음부터 설치하는 전 과정 — 사전 준비, 런타임 설치, kubeadm init/join, CNI 배포까지 단계별 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, kubeadm, Install, containerd, CKA, DevOps]

permalink: /kubernetes/kubernetes-install-kubeadm/

toc: true
toc_sticky: true

date: 2026-06-23
last_modified_at: 2026-06-23
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Install Kubernetes the kubeadm way 섹션을 기반으로 정리한 내용이에요.

---

## 🦥 kubeadm이란?

kubeadm은 K8s 공식 클러스터 설치 도구예요. 복잡한 K8s 컴포넌트 설치와 설정을 자동화해줘요.

| 항목 | 설명 |
|------|------|
| 역할 | K8s 클러스터 부트스트래핑 (init, join) |
| 범위 | Control Plane 구성, 인증서 생성, etcd 설정 |
| 안 하는 것 | OS 설정, Container Runtime 설치, CNI 설치 |
| CKA 시험 | **직접 출제됨** — kubeadm으로 클러스터 설치 문제 |

> 💡 kubeadm은 클러스터 **부트스트래핑**만 담당해요. 사전 준비(OS, 런타임)와 사후 설정(CNI)은 직접 해야 해요.

---

## 🦥 설치 전체 흐름

| 단계 | 작업 | 대상 노드 |
|------|------|----------|
| 1 | 사전 준비 (OS 설정) | 모든 노드 |
| 2 | Container Runtime 설치 | 모든 노드 |
| 3 | kubeadm, kubelet, kubectl 설치 | 모든 노드 |
| 4 | `kubeadm init` (클러스터 초기화) | Master만 |
| 5 | kubeconfig 설정 | Master만 |
| 6 | CNI 플러그인 배포 | Master에서 실행 |
| 7 | `kubeadm join` (워커 조인) | Worker만 |
| 8 | 설치 확인 | Master에서 실행 |

---

## 🦥 Step 1: 사전 준비 (모든 노드)

### 최소 요구 사항

| 항목 | Master | Worker |
|------|--------|--------|
| CPU | 2개 이상 | 1개 이상 |
| RAM | 2GB 이상 | 1GB 이상 |
| 디스크 | 20GB 이상 | 20GB 이상 |
| OS | Ubuntu 20.04/22.04 LTS | Ubuntu 20.04/22.04 LTS |
| 네트워크 | 노드 간 통신 가능 | 노드 간 통신 가능 |

### Swap 비활성화

K8s는 Swap이 활성화되어 있으면 kubelet이 정상 동작하지 않아요.

```bash
# Swap 즉시 비활성화
sudo swapoff -a

# 영구 비활성화 (/etc/fstab에서 swap 라인 주석 처리)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 확인
free -h    # Swap이 0이면 성공
```

### 커널 모듈 및 네트워크 설정

```bash
# 필요한 커널 모듈 로드
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 네트워크 브릿지 설정 (iptables가 브릿지 트래픽 처리)
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 적용
sudo sysctl --system
```

| 설정 | 이유 |
|------|------|
| `overlay` | Container Runtime의 Overlay 파일 시스템 |
| `br_netfilter` | Bridge 네트워크 트래픽을 iptables로 처리 |
| `ip_forward` | 노드 간 패킷 포워딩 (Pod 통신) |

---

## 🦥 Step 2: Container Runtime 설치 (모든 노드)

K8s 1.24부터 Docker(dockershim)는 제거되었어요. **containerd**를 사용해요.

### containerd 설치

```bash
# 의존 패키지 설치
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Docker 공식 GPG 키 추가
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker 리포지토리 추가 (containerd 포함)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# containerd 설치
sudo apt-get update
sudo apt-get install -y containerd.io
```

### containerd 설정

```bash
# 기본 설정 파일 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# SystemdCgroup 활성화 (중요!)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# containerd 재시작
sudo systemctl restart containerd
sudo systemctl enable containerd
```

> ⚠️ **CKA 필수**: `SystemdCgroup = true` 설정을 빠뜨리면 kubelet과 containerd의 cgroup 드라이버가 불일치하여 Pod이 정상 동작하지 않아요!

---

## 🦥 Step 3: kubeadm, kubelet, kubectl 설치 (모든 노드)

```bash
# K8s 패키지 리포지토리 설정
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# K8s GPG 키 추가 (v1.30 예시)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# K8s 리포지토리 추가
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# kubeadm, kubelet, kubectl 설치
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 버전 고정 (자동 업그레이드 방지)
sudo apt-mark hold kubelet kubeadm kubectl
```

| 패키지 | 역할 |
|--------|------|
| `kubeadm` | 클러스터 부트스트래핑 도구 |
| `kubelet` | 노드에서 Pod을 실행하는 에이전트 |
| `kubectl` | K8s CLI 클라이언트 |

> 💡 `apt-mark hold`로 버전을 고정해야 `apt upgrade` 시 실수로 K8s가 업그레이드되는 걸 방지할 수 있어요.

---

## 🦥 Step 4: 클러스터 초기화 (Master만)

```bash
# kubeadm init 실행
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP>
```

| 옵션 | 설명 |
|------|------|
| `--pod-network-cidr` | Pod 네트워크 대역 (CNI에 맞게 설정) |
| `--apiserver-advertise-address` | apiserver가 사용할 Master IP |
| `--control-plane-endpoint` | HA 구성 시 LB 주소 |
| `--kubernetes-version` | 특정 K8s 버전 지정 |

### init이 하는 일

| 순서 | 동작 |
|------|------|
| 1 | preflight checks (사전 검증) |
| 2 | CA 인증서 생성 (`/etc/kubernetes/pki/`) |
| 3 | kubeconfig 파일 생성 (`/etc/kubernetes/`) |
| 4 | Static Pod manifest 생성 (etcd, apiserver, scheduler, controller-manager) |
| 5 | kubelet 시작 → Static Pod 실행 |
| 6 | `kubeadm join` 토큰 출력 |

### init 성공 후 출력 예시

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxx
```

> ⚠️ **중요**: `kubeadm join` 명령어를 반드시 메모해두세요! 나중에 Worker 노드를 조인할 때 필요해요.

---

## 🦥 Step 5: kubeconfig 설정 (Master)

```bash
# 일반 사용자로 kubectl 사용 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 확인
kubectl get nodes
# NAME     STATUS     ROLES           AGE   VERSION
# master   NotReady   control-plane   1m    v1.30.0
```

노드가 **NotReady** 상태인 건 정상이에요. CNI 플러그인이 아직 없기 때문이에요.

---

## 🦥 Step 6: CNI 플러그인 배포 (Master에서 실행)

CNI를 설치해야 Pod 네트워크가 동작하고 노드가 Ready 상태가 돼요.

```bash
# Flannel 설치 (--pod-network-cidr=10.244.0.0/16과 매칭)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# 또는 Calico 설치
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 또는 Weave 설치
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

| CNI | `--pod-network-cidr` 값 |
|-----|-------------------------|
| Flannel | `10.244.0.0/16` (필수 지정) |
| Calico | `192.168.0.0/16` (기본값, 변경 가능) |
| Weave | 자동 할당 (지정 불필요) |

```bash
# CNI 설치 후 노드 상태 확인
kubectl get nodes
# NAME     STATUS   ROLES           AGE   VERSION
# master   Ready    control-plane   5m    v1.30.0
```

---

## 🦥 Step 7: Worker 노드 조인

Master에서 `kubeadm init` 후 출력된 join 명령어를 Worker 노드에서 실행해요.

```bash
# Worker 노드에서 실행 (root 또는 sudo)
sudo kubeadm join 192.168.1.10:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxx
```

### 토큰을 잊었을 때

```bash
# Master에서 새 토큰 생성
kubeadm token create --print-join-command

# 기존 토큰 목록
kubeadm token list
```

---

## 🦥 Step 8: 설치 확인

```bash
# 노드 상태 확인
kubectl get nodes -o wide

# 시스템 Pod 확인
kubectl get pods -n kube-system

# 모든 컴포넌트 상태 확인
kubectl get componentstatuses    # deprecated지만 참고용
kubectl get --raw='/readyz?verbose'
```

### 정상 상태 체크리스트

| 확인 항목 | 명령어 | 기대 결과 |
|----------|--------|----------|
| 모든 노드 Ready | `kubectl get nodes` | STATUS: Ready |
| CoreDNS 동작 | `kubectl get pods -n kube-system` | coredns Running |
| kube-proxy 동작 | `kubectl get pods -n kube-system` | kube-proxy Running |
| CNI 동작 | `kubectl get pods -n kube-system` | flannel/calico/weave Running |
| DNS 해석 | `kubectl run test --image=busybox --rm -it -- nslookup kubernetes` | 정상 응답 |

---

## 🦥 자주 발생하는 문제와 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| `kubeadm init` preflight 실패 | Swap 활성화, 포트 충돌 | swap 끄기, 포트 확인 |
| 노드 NotReady | CNI 미설치 | CNI 플러그인 배포 |
| kubelet 시작 실패 | cgroup 드라이버 불일치 | containerd `SystemdCgroup=true` 확인 |
| CoreDNS CrashLoopBackOff | 루프 감지 (호스트 DNS 문제) | `/etc/resolv.conf` 확인 |
| 토큰 만료 | 기본 24시간 유효 | `kubeadm token create` |
| join 실패 | 네트워크, 방화벽 | 6443 포트 열기, ping 확인 |

```bash
# kubelet 로그 확인 (문제 진단)
sudo journalctl -u kubelet -f

# kubeadm 초기화 실패 시 리셋
sudo kubeadm reset
# → 다시 init부터 수행
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **kubeadm** | K8s 공식 클러스터 부트스트래핑 도구, CKA 출제 |
| **사전 준비** | Swap 끄기, 커널 모듈, ip_forward 활성화 |
| **Container Runtime** | containerd 설치, `SystemdCgroup=true` 필수 |
| **kubeadm init** | Master 초기화, 인증서/kubeconfig/Static Pod 생성 |
| **CNI 배포** | init 후 반드시 설치해야 노드가 Ready |
| **kubeadm join** | Worker 노드 클러스터 합류, 토큰 필요 |
| **토큰 재생성** | `kubeadm token create --print-join-command` |
| **문제 해결** | `journalctl -u kubelet`, `kubeadm reset` |

이것으로 kubeadm 설치 가이드를 마무리할게요! 사전 준비부터 클러스터 초기화, Worker 조인, 검증까지 — CKA 시험에서 직접 수행해야 하는 실습 내용을 모두 다뤘어요. 🦥
