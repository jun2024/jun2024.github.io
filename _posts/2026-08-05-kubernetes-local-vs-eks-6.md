---
title: "[Kubernetes] 로컬 K8s vs EKS (6) — 업그레이드 & 유지보수: kubeadm upgrade vs EKS 업그레이드"
excerpt: "K8s 클러스터 업그레이드가 이렇게 다르다! kubeadm upgrade plan/apply vs EKS Control Plane/Node Group 업그레이드, 백업 전략 비교 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, EKS, AWS, Upgrade, Maintenance, Velero, DevOps]

permalink: /kubernetes/local-vs-eks-6/

toc: true
toc_sticky: true

date: 2026-08-05
last_modified_at: 2026-08-05
---

> 이전 포스트 [로컬 vs EKS (5)](/kubernetes/local-vs-eks-5/)에서 인증 & 권한 차이를 다뤘어요!

---

## 🦥 업그레이드 비교 — 관리 부담의 핵심 차이

K8s 클러스터 업그레이드는 운영에서 가장 긴장되는 작업이에요. 로컬과 EKS에서 방식이 크게 달라요.

| 항목 | 로컬 (kubeadm) | EKS |
|------|---------------|-----|
| Control Plane | `kubeadm upgrade apply` | **AWS가 자동 (클릭/CLI)** |
| Worker Node | drain → 업그레이드 → uncordon | **Node Group 롤링 업데이트** |
| 다운타임 | 주의 필요 | 거의 없음 (HA 기본) |
| 롤백 | 어려움 | Control Plane 롤백 불가 |
| 건너뛰기 | 마이너 버전 1개씩만 | 동일 |

---

## 🦥 로컬 K8s — kubeadm upgrade 복습

[클러스터 유지보수 (1)](/kubernetes/kubernetes-cluster-maintenance-1/)에서 자세히 다뤘어요. 핵심 흐름만 요약하면 이래요.

### Control Plane 업그레이드

```bash
# 1. 업그레이드 계획 확인
kubeadm upgrade plan

# 2. kubeadm 업그레이드
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.31.0-1.1
sudo apt-mark hold kubeadm

# 3. Control Plane 업그레이드 적용
sudo kubeadm upgrade apply v1.31.0

# 4. kubelet, kubectl 업그레이드
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Worker Node 업그레이드

```bash
# Master에서: Worker drain
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# Worker에서: kubeadm, kubelet 업그레이드
sudo apt-get install -y kubeadm=1.31.0-1.1
sudo kubeadm upgrade node
sudo apt-get install -y kubelet=1.31.0-1.1
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Master에서: Worker uncordon
kubectl uncordon worker-1
```

| 직접 해야 하는 것 | 위험 요소 |
|----------------|----------|
| 버전 호환성 확인 | 패키지 버전 불일치 |
| 노드별 순차 업그레이드 | drain 중 서비스 영향 |
| 인증서 갱신 확인 | 인증서 만료 시 클러스터 장애 |
| etcd 백업 | 업그레이드 실패 시 복구 필요 |

---

## 🦥 EKS — 업그레이드 프로세스

### Control Plane 업그레이드

EKS Control Plane 업그레이드는 **AWS가 처리**해요. 클릭 한 번이에요.

```bash
# eksctl로 업그레이드
eksctl upgrade cluster --name my-cluster --version 1.31 --approve

# AWS CLI로 업그레이드
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.31
```

| 단계 | AWS가 하는 일 |
|------|-------------|
| 1 | 새 버전의 Control Plane 프로비저닝 |
| 2 | 기존 트래픽을 새 Control Plane으로 전환 |
| 3 | 이전 Control Plane 제거 |
| 4 | 전체 과정 약 20~40분 |

> 💡 EKS Control Plane 업그레이드는 **무중단**이에요. apiserver가 잠깐 불안정할 수 있지만 기존 Pod은 계속 동작해요.

### Node Group 업그레이드

Control Plane 업그레이드 후 **Node Group도 별도로 업그레이드**해야 해요.

```bash
# Managed Node Group 업그레이드
eksctl upgrade nodegroup \
  --cluster my-cluster \
  --name my-nodes \
  --kubernetes-version 1.31

# AWS CLI
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes
```

### Managed Node Group 롤링 업데이트 과정

| 단계 | 동작 |
|------|------|
| 1 | 새 버전의 EC2 Launch Template 생성 |
| 2 | 새 노드 프로비저닝 |
| 3 | 기존 노드 drain (Pod 이동) |
| 4 | 기존 노드 종료 |
| 5 | 모든 노드가 교체될 때까지 반복 |

```bash
# 업그레이드 상태 확인
aws eks describe-update \
  --name my-cluster \
  --update-id <update-id>

# 노드 버전 확인
kubectl get nodes -o wide
```

### 업그레이드 전략 설정

```yaml
# eksctl 설정 파일에서 업그레이드 전략 지정
managedNodeGroups:
  - name: my-nodes
    updateConfig:
      maxUnavailable: 1              # 동시에 업데이트되는 최대 노드 수
      # 또는
      maxUnavailablePercentage: 25   # 동시에 업데이트되는 최대 비율
```

---

## 🦥 Add-on 업그레이드

EKS에서는 Add-on도 별도로 업그레이드해야 해요.

```bash
# Add-on 현재 버전 확인
aws eks describe-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni

# Add-on 업그레이드
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

| Add-on | 업그레이드 시점 |
|--------|--------------|
| vpc-cni | Control Plane 업그레이드 후 |
| coredns | Control Plane 업그레이드 후 |
| kube-proxy | Control Plane 업그레이드 후 |
| ebs-csi-driver | 필요 시 |

### EKS 업그레이드 전체 순서

| 순서 | 작업 | 소요 시간 |
|------|------|----------|
| 1 | Control Plane 업그레이드 | 20~40분 |
| 2 | Add-on 업그레이드 | 5~10분 |
| 3 | Node Group 업그레이드 | 노드 수에 비례 |
| 4 | 애플리케이션 호환성 확인 | - |

---

## 🦥 백업 & 복구 전략 비교

### 로컬 K8s — etcd 직접 백업

```bash
# etcd 스냅샷 백업 (CKA에서 배운 방법)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 복구
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-from-backup
```

### EKS — Velero + S3

EKS에서는 etcd에 접근할 수 없으므로 **K8s 리소스 레벨**에서 백업해요.

```bash
# Velero 설치
velero install \
  --provider aws \
  --bucket my-velero-bucket \
  --backup-location-config region=ap-northeast-2 \
  --snapshot-location-config region=ap-northeast-2 \
  --plugins velero/velero-plugin-for-aws:v1.9.0

# 백업 생성
velero backup create my-backup \
  --include-namespaces my-app,production

# 스케줄 백업 (매일 자정)
velero schedule create daily-backup \
  --schedule="0 0 * * *" \
  --include-namespaces my-app

# 복구
velero restore create --from-backup my-backup
```

| 항목 | etcd 백업 (로컬) | Velero (EKS) |
|------|-----------------|-------------|
| 범위 | 클러스터 전체 상태 | 선택적 (Namespace, Label) |
| 저장소 | 로컬 디스크 | S3 |
| PV 백업 | 별도 처리 | EBS 스냅샷 연동 가능 |
| 복구 대상 | 같은 클러스터만 | 다른 클러스터에도 복구 가능 |
| 스케줄링 | cron으로 직접 | Velero Schedule |

---

## 🦥 인증서 관리

| 항목 | 로컬 K8s | EKS |
|------|---------|-----|
| CA 인증서 | `/etc/kubernetes/pki/ca.crt` (10년) | AWS 관리 |
| 컴포넌트 인증서 | 1년 유효, 수동 갱신 필요 | AWS 자동 갱신 |
| 갱신 명령 | `kubeadm certs renew all` | 불필요 |
| 만료 확인 | `kubeadm certs check-expiration` | 불필요 |

```bash
# 로컬: 인증서 만료 확인 (정기적으로 해야 함)
kubeadm certs check-expiration

# 로컬: 인증서 갱신
kubeadm certs renew all
sudo systemctl restart kubelet
```

> 💡 **로컬 K8s의 흔한 장애**: 인증서 갱신을 깜빡하면 1년 후 갑자기 `kubectl` 명령이 안 돼요. EKS에서는 이런 걱정이 없어요.

---

## 🦥 EKS 버전 지원 정책

| 정책 | 설명 |
|------|------|
| 지원 기간 | 마이너 버전 출시 후 **14개월** 표준 지원 |
| 연장 지원 | 표준 지원 후 **12개월** 추가 (추가 비용) |
| 자동 업그레이드 | 연장 지원 끝나면 **강제 업그레이드** |
| 건너뛰기 | 마이너 버전 1개씩만 (1.29 → 1.30 → 1.31) |

> ⚠️ **실무 주의**: EKS 버전 지원이 끝나면 AWS가 강제로 업그레이드해요. 호환성 문제가 생길 수 있으므로 미리 계획적으로 업그레이드하세요.

---

## 🦥 정리

| 주제 | 로컬 K8s | EKS |
|------|---------|-----|
| **Control Plane 업그레이드** | kubeadm upgrade (수동) | AWS 관리 (CLI/Console) |
| **Worker 업그레이드** | drain → 업그레이드 → uncordon | Node Group 롤링 업데이트 |
| **백업** | etcd snapshot | Velero + S3 |
| **인증서** | 수동 갱신 (1년) | AWS 자동 갱신 |
| **Add-on** | 직접 관리 | EKS Add-on 버전 관리 |
| **버전 지원** | 커뮤니티 지원 | 14개월 + 12개월 연장 |

다음 포스트에서는 **모니터링 & 로깅 비교**로 시리즈를 마무리할게요! 🦥
