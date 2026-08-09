---
title: "[Kubernetes] 로컬 K8s vs EKS (2) — 클러스터 설치: kubeadm vs eksctl & Terraform"
excerpt: "클러스터 설치 방법이 이렇게 다르다! kubeadm init/join vs eksctl create, VPC CNI, EKS Add-on 관리까지 비교 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, EKS, AWS, kubeadm, eksctl, Terraform, DevOps]

permalink: /kubernetes/local-vs-eks-2/

toc: true
toc_sticky: true

date: 2026-07-24
last_modified_at: 2026-07-24
---

> 이전 포스트 [로컬 vs EKS (1)](/kubernetes/local-vs-eks-1/)에서 아키텍처 차이를 다뤘어요!

---

## 🦥 설치 방법 한눈에 비교

| 항목 | 로컬 (kubeadm) | EKS (eksctl) | EKS (Terraform) |
|------|---------------|-------------|-----------------|
| 난이도 | 높음 | 낮음 | 중간 |
| 자동화 | 수동 스크립트 | CLI 한 줄 | IaC 코드 |
| 재현성 | 스크립트 의존 | 제한적 | 완전한 IaC |
| 프로덕션 적합 | 가능 (관리 부담 큼) | 빠른 시작 | 권장 |
| 소요 시간 | 30분~1시간 | 15~20분 | 20~30분 |

---

## 🦥 로컬 K8s — kubeadm 설치 복습

kubeadm 설치 과정은 [kubeadm 설치 가이드](/kubernetes/kubernetes-install-kubeadm/)에서 자세히 다뤘어요. 핵심 흐름만 요약하면 이래요.

```bash
# 1. 사전 준비 (모든 노드)
sudo swapoff -a
sudo modprobe overlay br_netfilter
# sysctl 설정...

# 2. containerd 설치 (모든 노드)
sudo apt-get install -y containerd.io
# SystemdCgroup = true 설정...

# 3. kubeadm/kubelet/kubectl 설치 (모든 노드)
sudo apt-get install -y kubelet kubeadm kubectl

# 4. Master 초기화
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# 5. kubeconfig 설정
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config

# 6. CNI 설치
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# 7. Worker 조인
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

| 직접 해야 하는 것 | 설명 |
|----------------|------|
| OS 설정 | swap, 커널 모듈, sysctl |
| Container Runtime | containerd 설치 및 설정 |
| K8s 패키지 | kubeadm, kubelet, kubectl |
| CNI 플러그인 | Flannel, Calico 등 직접 선택/설치 |
| 인증서 | kubeadm이 자동 생성하지만 갱신은 직접 |

---

## 🦥 EKS — eksctl로 설치

### eksctl이란?

eksctl은 EKS 클러스터를 CLI 한 줄로 만들 수 있는 공식 도구예요.

```bash
# 가장 간단한 클러스터 생성
eksctl create cluster \
  --name my-cluster \
  --region ap-northeast-2 \
  --version 1.30 \
  --nodegroup-name my-nodes \
  --node-type t3.medium \
  --nodes 3
```

이 한 줄이 자동으로 하는 일이에요.

| 순서 | 자동 생성 리소스 |
|------|----------------|
| 1 | VPC, 서브넷 (Public + Private), IGW, NAT Gateway |
| 2 | EKS 클러스터 (Control Plane) |
| 3 | IAM Role (클러스터, 노드 그룹) |
| 4 | Security Group |
| 5 | Managed Node Group (EC2 인스턴스) |
| 6 | kubeconfig 자동 업데이트 |

> 💡 kubeadm은 7단계를 직접 수행하지만, eksctl은 **한 줄**로 끝나요. 대신 내부에서 무슨 일이 일어나는지 이해하기 어려워요.

### eksctl 설정 파일

복잡한 설정은 YAML 파일로 관리해요.

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: ap-northeast-2
  version: "1.30"

vpc:
  cidr: 10.0.0.0/16

managedNodeGroups:
  - name: app-nodes
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 1
    maxSize: 5
    volumeSize: 50
    labels:
      role: app
    tags:
      environment: production

  - name: gpu-nodes
    instanceType: g4dn.xlarge
    desiredCapacity: 1
    labels:
      role: gpu
```

```bash
# 설정 파일로 클러스터 생성
eksctl create cluster -f cluster-config.yaml
```

---

## 🦥 EKS — Terraform으로 설치

프로덕션에서는 **Terraform**으로 EKS를 구성하는 게 권장돼요. 인프라를 코드로 관리(IaC)할 수 있어요.

```hcl
# main.tf — EKS 클러스터 기본 구성
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.30"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Control Plane 엔드포인트 접근 설정
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # Managed Node Group
  eks_managed_node_groups = {
    app = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 5
      desired_size   = 3

      labels = {
        role = "app"
      }
    }
  }

  # EKS Add-ons
  cluster_addons = {
    coredns    = { most_recent = true }
    kube-proxy = { most_recent = true }
    vpc-cni    = { most_recent = true }
  }
}
```

| eksctl vs Terraform | eksctl | Terraform |
|--------------------|--------|-----------|
| 학습 곡선 | 낮음 | 중간 |
| 상태 관리 | 없음 (CloudFormation) | terraform.tfstate |
| 코드 리뷰 | 어려움 | PR로 인프라 리뷰 가능 |
| 모듈화 | 제한적 | 자유로운 모듈 구성 |
| 다른 AWS 리소스 통합 | 별도 관리 | 같은 코드에서 관리 |

---

## 🦥 CNI 차이 — 가장 큰 네트워크 차이

### 로컬 K8s — Overlay 네트워크

로컬에서는 Flannel, Calico 등 **Overlay 네트워크** CNI를 사용해요. Pod IP는 클러스터 내부에서만 유효해요.

```
Node IP: 192.168.1.10          Node IP: 192.168.1.11
┌─────────────────┐           ┌─────────────────┐
│ Pod: 10.244.0.5 │──Overlay──│ Pod: 10.244.1.3 │
│ Pod: 10.244.0.6 │  터널링   │ Pod: 10.244.1.4 │
└─────────────────┘           └─────────────────┘
```

Pod IP(10.244.x.x)는 **노드 외부에서 직접 접근할 수 없어요**.

### EKS — AWS VPC CNI

EKS의 기본 CNI는 **AWS VPC CNI**예요. Pod에 **VPC의 실제 IP**가 할당돼요.

```
VPC CIDR: 10.0.0.0/16
Node ENI: 10.0.1.10           Node ENI: 10.0.2.10
┌─────────────────┐           ┌─────────────────┐
│ Pod: 10.0.1.25  │──VPC 내부──│ Pod: 10.0.2.30 │
│ Pod: 10.0.1.26  │  직접통신  │ Pod: 10.0.2.31 │
└─────────────────┘           └─────────────────┘
```

| 항목 | Overlay CNI (Flannel 등) | VPC CNI (EKS) |
|------|------------------------|---------------|
| Pod IP | 클러스터 내부 전용 | VPC 실제 IP |
| 외부 접근 | NAT 필요 | VPC 내에서 직접 접근 가능 |
| 성능 | 캡슐화 오버헤드 | 네이티브 성능 |
| IP 제한 | 거의 없음 | **EC2 ENI 제한에 따름** |
| 보안 그룹 | Pod 레벨 불가 | **Pod 레벨 SG 가능** |

### VPC CNI의 IP 제한

VPC CNI에서 각 EC2 인스턴스가 가질 수 있는 Pod 수는 **ENI(Elastic Network Interface) 수 × IP 수**로 제한돼요.

| 인스턴스 타입 | 최대 ENI | ENI당 IP | 최대 Pod 수 |
|-------------|---------|---------|-----------|
| t3.micro | 2 | 2 | 4 |
| t3.medium | 3 | 6 | 17 |
| t3.large | 3 | 12 | 35 |
| m5.large | 3 | 10 | 29 |
| m5.xlarge | 4 | 15 | 58 |

> ⚠️ **실무 주의**: t3.micro는 최대 4개 Pod만 실행 가능해요. 시스템 Pod(kube-proxy, CoreDNS 등)을 빼면 사용자 Pod은 1~2개뿐이에요.

---

## 🦥 EKS Add-on 관리

로컬 K8s에서는 CoreDNS, kube-proxy를 직접 설치하지만, EKS에서는 **Add-on**으로 관리해요.

| Add-on | 로컬 K8s | EKS |
|--------|---------|-----|
| CoreDNS | CNI와 함께 자동 설치 | EKS Add-on으로 관리 |
| kube-proxy | kubeadm이 자동 설치 | EKS Add-on으로 관리 |
| CNI | 직접 선택/설치 | VPC CNI Add-on (기본) |
| EBS CSI | 해당 없음 | Add-on으로 설치 |

```bash
# EKS Add-on 목록 확인
aws eks list-addons --cluster-name my-cluster

# Add-on 설치
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789012:role/EBS-CSI-Role

# eksctl로 Add-on 관리
eksctl get addons --cluster my-cluster
```

---

## 🦥 정리

| 주제 | 로컬 K8s | EKS |
|------|---------|-----|
| **설치 도구** | kubeadm (7단계 수동) | eksctl 한 줄 / Terraform IaC |
| **CNI** | Flannel, Calico (Overlay) | VPC CNI (VPC 실제 IP) |
| **Pod IP** | 클러스터 내부 전용 | VPC 내에서 직접 접근 가능 |
| **Pod 수 제한** | 노드당 110개 (기본) | EC2 ENI 제한에 따름 |
| **Add-on** | 직접 설치/관리 | EKS Add-on으로 버전 관리 |
| **IaC** | Ansible, 스크립트 | Terraform (권장) |

다음 포스트에서는 **네트워킹 비교 — Service 노출, Ingress, DNS 차이**를 다룰게요! 🦥
