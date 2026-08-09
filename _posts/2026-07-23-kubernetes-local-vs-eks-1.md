---
title: "[Kubernetes] 로컬 K8s vs EKS (1) — 아키텍처 비교: 무엇이 다를까?"
excerpt: "같은 K8s인데 왜 운영 방식이 다를까? 로컬 클러스터와 AWS EKS의 아키텍처, Control Plane 관리, etcd 접근, 비용 구조를 비교 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, EKS, AWS, Architecture, DevOps]

permalink: /kubernetes/local-vs-eks-1/

toc: true
toc_sticky: true

date: 2026-07-23
last_modified_at: 2026-07-23
---

> 이 시리즈는 CKA에서 배운 K8s 개념이 **AWS EKS 실무**에서 어떻게 달라지는지 비교하는 내용이에요.

---

## 🦥 같은 K8s, 다른 운영

kubeadm으로 직접 설치한 클러스터와 EKS는 둘 다 Kubernetes예요. `kubectl get pods`도 동일하게 동작해요. 하지만 **누가, 무엇을 관리하는가**에서 큰 차이가 있어요.

| 구분 | 로컬 K8s (kubeadm) | AWS EKS |
|------|-------------------|---------|
| Control Plane | **직접 관리** | **AWS가 관리** |
| Worker Node | 직접 프로비저닝 | EC2, Fargate 선택 |
| etcd | 직접 접근 가능 | 접근 불가 (AWS 관리) |
| 인증서 관리 | 직접 갱신 | AWS가 자동 갱신 |
| 업그레이드 | kubeadm upgrade | 콘솔/eksctl 클릭 |
| 비용 | 서버 비용만 | 클러스터당 $0.10/hr + 노드 비용 |

---

## 🦥 Control Plane 비교

### 로컬 K8s — 모든 걸 직접 관리

kubeadm으로 설치하면 Master 노드에 모든 Control Plane 컴포넌트가 **Static Pod**으로 실행돼요.

```bash
# Master 노드에서 직접 확인 가능
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

# etcd 직접 접근 가능
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

| 할 수 있는 것 | 해야 하는 것 |
|-------------|------------|
| etcd 직접 백업/복구 | 인증서 갱신 (1년마다) |
| apiserver 설정 변경 | Control Plane 모니터링 |
| scheduler/controller 커스터마이징 | 보안 패치 적용 |
| 클러스터 디버깅 전체 | 고가용성(HA) 직접 구성 |

### EKS — Control Plane은 AWS 영역

EKS에서는 Control Plane이 **AWS가 관리하는 별도 VPC**에서 실행돼요. 사용자는 apiserver endpoint를 통해서만 접근할 수 있어요.

```bash
# EKS 클러스터 생성
eksctl create cluster --name my-cluster --region ap-northeast-2

# kubeconfig 자동 설정
aws eks update-kubeconfig --name my-cluster --region ap-northeast-2

# kubectl은 동일하게 동작
kubectl get nodes
kubectl get pods -A
```

| 할 수 없는 것 | 안 해도 되는 것 |
|-------------|--------------|
| Master 노드 SSH 접근 | etcd 백업/복구 |
| etcd 직접 조작 | 인증서 갱신 |
| Static Pod manifest 수정 | Control Plane HA 구성 |
| apiserver 플래그 일부 변경 | Control Plane 패치 |

> 💡 **핵심 차이**: 로컬에서는 `etcdctl`로 직접 백업하지만, EKS에서는 **AWS가 자동으로 etcd를 백업**해요. CKA에서 배운 etcd 백업/복구는 EKS에서는 사용하지 않아요.

---

## 🦥 Worker Node 비교

### 로컬 K8s — VM 또는 물리 서버

```bash
# Worker 노드 조인
kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash <hash>
```

노드 추가/제거, OS 패치, 커널 업그레이드를 모두 직접 해야 해요.

### EKS — 3가지 선택지

| 옵션 | 설명 | 관리 부담 |
|------|------|----------|
| **Managed Node Group** | EC2 인스턴스 그룹, AWS가 프로비저닝 | 중간 |
| **Self-managed Node** | EC2를 직접 관리, kubeadm join과 유사 | 높음 |
| **Fargate** | 서버리스, Pod 단위로 자동 프로비저닝 | 최소 |

```bash
# Managed Node Group 생성
eksctl create nodegroup \
  --cluster my-cluster \
  --name my-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 5
```

### Fargate — 서버리스 K8s

```bash
# Fargate Profile 생성
eksctl create fargateprofile \
  --cluster my-cluster \
  --name my-fargate \
  --namespace my-app
```

| 구분 | EC2 Node Group | Fargate |
|------|---------------|---------|
| 인프라 관리 | EC2 인스턴스 관리 필요 | 완전 서버리스 |
| 과금 | EC2 인스턴스 시간당 | vCPU + 메모리 사용량 |
| DaemonSet | 지원 | **미지원** |
| GPU | 지원 | 미지원 |
| 스토리지 | EBS, EFS, hostPath | EFS만 지원 |
| 적합한 경우 | 상시 워크로드, DaemonSet 필요 | 배치, 간헐적 워크로드 |

> ⚠️ **Fargate 제약**: DaemonSet을 사용할 수 없어요. Fluentd, Datadog Agent 같은 노드 레벨 도구는 Fargate에서 동작하지 않아요. 사이드카 패턴으로 대체해야 해요.

---

## 🦥 etcd 접근 차이

이 부분이 로컬과 EKS의 **가장 근본적인 차이** 중 하나예요.

| 항목 | 로컬 K8s | EKS |
|------|---------|-----|
| etcd 위치 | Master 노드 내부 | AWS 관리형 (접근 불가) |
| 백업 | `etcdctl snapshot save` | AWS 자동 백업 |
| 복구 | `etcdctl snapshot restore` | AWS 지원 요청 또는 클러스터 재생성 |
| 데이터 암호화 | 직접 EncryptionConfiguration | KMS 연동으로 간편 설정 |

```bash
# 로컬: etcd 직접 백업 (CKA에서 배운 방법)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db ...

# EKS: etcd 접근 자체가 불가
# 대신 K8s 리소스 레벨에서 백업
# Velero 등의 도구 사용
velero backup create my-backup --include-namespaces my-app
```

> 💡 EKS에서는 **Velero**로 K8s 리소스를 S3에 백업하는 게 일반적이에요. etcd 레벨 백업은 AWS 영역이에요.

---

## 🦥 비용 구조 비교

### 로컬 K8s

| 비용 항목 | 설명 |
|----------|------|
| 서버/VM 비용 | 물리 서버 또는 클라우드 VM |
| 운영 인력 | Control Plane 관리 인건비 |
| 네트워크 | 자체 네트워크 장비 |
| 스토리지 | NFS, Ceph 등 자체 구축 |

### EKS

| 비용 항목 | 금액 (2024 기준) |
|----------|----------------|
| EKS 클러스터 | $0.10/hr ($73/월) |
| EC2 Worker 노드 | 인스턴스 타입별 과금 |
| Fargate | vCPU $0.04048/hr + Memory $0.004445/hr/GB |
| 데이터 전송 | AZ 간, 인터넷 아웃바운드 |
| EBS/EFS | 스토리지 사용량 |
| ALB/NLB | 로드밸런서 시간 + LCU |

> 💡 **실무 팁**: 소규모 팀이라면 EKS의 $73/월 고정 비용이 부담될 수 있어요. 하지만 Control Plane 운영 인건비를 고려하면 EKS가 더 경제적인 경우가 많아요.

---

## 🦥 언제 어떤 걸 선택할까?

| 상황 | 추천 |
|------|------|
| CKA 학습/실습 | 로컬 (kubeadm) |
| K8s 내부 동작 이해 | 로컬 (kubeadm) |
| 프로덕션 서비스 (AWS) | EKS |
| 온프레미스 규정 준수 | 로컬 (kubeadm, kubespray) |
| 빠른 프로토타이핑 | EKS + Fargate |
| GPU/ML 워크로드 | EKS + GPU 인스턴스 |
| 멀티 클라우드 | 로컬 또는 각 클라우드 관리형 |

---

## 🦥 정리

| 주제 | 로컬 K8s | EKS |
|------|---------|-----|
| **Control Plane** | 직접 관리 (Static Pod) | AWS 완전 관리 |
| **etcd** | 직접 백업/복구 | 접근 불가, AWS 자동 백업 |
| **Worker** | VM 직접 관리 | Managed Node Group / Fargate |
| **인증서** | 수동 갱신 (1년) | 자동 갱신 |
| **HA** | 직접 구성 (3 Master) | 기본 제공 (3 AZ) |
| **비용** | 서버 + 인건비 | $73/월 + 노드 비용 |

다음 포스트에서는 **클러스터 설치 방법 비교 — kubeadm vs eksctl/Terraform**을 다룰게요! 🦥
