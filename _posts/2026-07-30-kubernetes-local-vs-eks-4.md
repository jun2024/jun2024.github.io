---
title: "[Kubernetes] 로컬 K8s vs EKS (4) — 스토리지: hostPath/NFS vs EBS/EFS CSI"
excerpt: "PV/PVC는 동일, 백엔드가 다르다! 로컬 hostPath/NFS vs AWS EBS/EFS, StorageClass, CSI Driver 설치와 IRSA 연동까지 비교 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, EKS, AWS, Storage, EBS, EFS, CSI, DevOps]

permalink: /kubernetes/local-vs-eks-4/

toc: true
toc_sticky: true

date: 2026-07-30
last_modified_at: 2026-07-30
---

> 이전 포스트 [로컬 vs EKS (3)](/kubernetes/local-vs-eks-3/)에서 네트워킹 차이를 다뤘어요!

---

## 🦥 K8s 스토리지 — 공통점과 차이점

PV, PVC, StorageClass 개념은 로컬이든 EKS든 **동일**해요. CKA에서 배운 내용이 그대로 적용돼요. 달라지는 건 **백엔드 스토리지와 CSI Driver**예요.

| 항목 | 로컬 K8s | EKS |
|------|---------|-----|
| PV/PVC 개념 | 동일 | 동일 |
| StorageClass | 동일 | 동일 |
| 사용할 수 있는 백엔드 | hostPath, NFS, Ceph 등 | **EBS, EFS, FSx** |
| CSI Driver | 필요에 따라 설치 | **직접 설치 필요 + IAM 연동** |
| 동적 프로비저닝 | StorageClass로 동일 | StorageClass로 동일 |

---

## 🦥 로컬 K8s 스토리지

### hostPath — 가장 간단한 방법

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/app
    type: DirectoryOrCreate
```

| 장점 | 한계 |
|------|------|
| 설정 간단 | 노드 종속 (다른 노드에서 접근 불가) |
| 추가 설치 없음 | 멀티 노드 환경에 부적합 |

### NFS — 멀티 노드 공유 스토리지

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany          # 여러 노드에서 동시 마운트
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

| 장점 | 한계 |
|------|------|
| 멀티 노드 공유 | NFS 서버 직접 구축/관리 |
| ReadWriteMany 지원 | 성능 제한 (네트워크 의존) |

---

## 🦥 EKS 스토리지 — EBS

### EBS (Elastic Block Store)

EBS는 AWS의 블록 스토리지로, **단일 노드**에 마운트돼요. 로컬의 `hostPath`와 비슷하지만 노드가 아닌 **AZ(가용 영역)에 종속**이에요.

| 특징 | 설명 |
|------|------|
| Access Mode | ReadWriteOnce (단일 노드) |
| 영속성 | Pod/노드 삭제 후에도 데이터 유지 |
| 성능 | gp3, io2 등 타입별 IOPS 선택 |
| 제한 | 같은 AZ의 노드에서만 마운트 가능 |

### EBS CSI Driver 설치

EKS에서 EBS를 사용하려면 **EBS CSI Driver**를 직접 설치해야 해요.

```bash
# 1. IRSA(IAM Role for Service Account) 생성
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# 2. EBS CSI Driver Add-on 설치
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789012:role/AmazonEKS_EBS_CSI_DriverRole
```

> ⚠️ **EKS 핵심**: EBS CSI Driver는 기본 설치가 아니에요! 직접 설치하지 않으면 PVC가 Pending 상태로 멈춰요.

### EBS StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3                    # EBS 볼륨 타입
  iops: "3000"                 # 기본 3000 IOPS
  throughput: "125"            # 기본 125 MB/s
  encrypted: "true"            # 암호화
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer    # Pod 스케줄 후 프로비저닝
reclaimPolicy: Delete
```

```yaml
# PVC (로컬과 동일한 방식)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 20Gi
```

| EBS 타입 | IOPS | 처리량 | 용도 |
|---------|------|--------|------|
| gp3 | 3,000 (기본) | 125 MB/s | 범용 (권장) |
| gp2 | 용량 비례 | - | 범용 (구버전) |
| io2 | 최대 64,000 | 1,000 MB/s | 고성능 DB |
| st1 | - | 500 MB/s | 빅데이터, 로그 |

> 💡 **`WaitForFirstConsumer`가 중요한 이유**: EBS는 AZ에 종속이에요. `Immediate`로 설정하면 볼륨이 먼저 생성되고, Pod이 다른 AZ에 스케줄되면 마운트 실패해요. `WaitForFirstConsumer`는 Pod이 스케줄될 AZ에서 볼륨을 생성해요.

---

## 🦥 EKS 스토리지 — EFS

### EFS (Elastic File System)

EFS는 AWS의 NFS 호환 파일 시스템으로, **여러 노드에서 동시 마운트**할 수 있어요. 로컬의 NFS와 대응돼요.

| 항목 | NFS (로컬) | EFS (EKS) |
|------|-----------|-----------|
| Access Mode | ReadWriteMany | ReadWriteMany |
| 관리 | NFS 서버 직접 구축 | AWS 완전 관리형 |
| AZ 제한 | 없음 | 없음 (Multi-AZ) |
| 성능 모드 | 서버 스펙 의존 | Bursting / Provisioned |
| 과금 | 서버 비용 | 사용량 기반 (GB/월) |

### EFS CSI Driver 설치

```bash
# 1. IRSA 생성
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace kube-system \
  --name efs-csi-controller-sa \
  --role-name AmazonEKS_EFS_CSI_DriverRole \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
  --approve

# 2. EFS CSI Driver Add-on 설치
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-efs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789012:role/AmazonEKS_EFS_CSI_DriverRole
```

### EFS StorageClass와 PV

```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap         # Access Point 모드
  fileSystemId: fs-0123456789abcdef
  directoryPerms: "700"
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-efs-pvc
spec:
  accessModes:
    - ReadWriteMany                 # 여러 Pod에서 동시 마운트
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi                  # EFS는 실제 제한 없음 (elastic)
```

> 💡 EFS는 **Elastic**이라서 `storage: 5Gi`를 지정해도 실제로는 무제한이에요. 사용한 만큼만 과금돼요.

---

## 🦥 EBS vs EFS 선택 기준

| 기준 | EBS | EFS |
|------|-----|-----|
| Access Mode | RWO (단일 노드) | RWX (멀티 노드) |
| 성능 | 높음 (블록 스토리지) | 중간 (네트워크 파일 시스템) |
| 비용 | 저렴 ($0.08/GB/월 gp3) | 비쌈 ($0.30/GB/월) |
| AZ 제한 | **있음** (같은 AZ만) | 없음 |
| Fargate | 미지원 | **지원** |
| 사용 사례 | DB, 단일 Pod 스토리지 | 공유 파일, CMS, ML 데이터 |

### 스토리지 선택 플로우

| 질문 | Yes → | No → |
|------|-------|------|
| 여러 Pod이 동시에 읽기/쓰기? | EFS | 다음 질문 |
| Fargate에서 사용? | EFS | 다음 질문 |
| 높은 IOPS 필요? | EBS (io2) | 다음 질문 |
| 비용 최적화 중요? | EBS (gp3) | EFS |

---

## 🦥 IRSA — CSI Driver의 IAM 연동

EKS에서 CSI Driver가 AWS 리소스(EBS, EFS)를 생성/삭제하려면 **IAM 권한**이 필요해요. **IRSA(IAM Roles for Service Accounts)**로 Pod에 IAM Role을 부여해요.

```
Pod (ServiceAccount) → OIDC Provider → AWS STS → IAM Role → AWS API
```

| 단계 | 설명 |
|------|------|
| 1 | EKS 클러스터에 OIDC Provider 연결 |
| 2 | IAM Role 생성 (Trust Policy에 OIDC 추가) |
| 3 | K8s ServiceAccount에 Role ARN 지정 |
| 4 | Pod이 해당 ServiceAccount로 실행 → 자동 인증 |

> 💡 로컬 K8s에는 IRSA 개념이 없어요. 이건 EKS 특유의 AWS-K8s 통합 메커니즘이에요. [다음 포스트](/kubernetes/local-vs-eks-5/)에서 IRSA를 자세히 다룰게요.

---

## 🦥 정리

| 주제 | 로컬 K8s | EKS |
|------|---------|-----|
| **블록 스토리지** | hostPath (노드 종속) | EBS (AZ 종속, 고성능) |
| **공유 스토리지** | NFS (직접 구축) | EFS (관리형, Multi-AZ) |
| **CSI Driver** | 필요 시 설치 | **직접 설치 + IRSA 필수** |
| **StorageClass** | local-path, nfs | ebs-gp3, efs-sc |
| **동적 프로비저닝** | StorageClass | StorageClass (동일) |
| **WaitForFirstConsumer** | 선택 | EBS에서 **필수** |

다음 포스트에서는 **인증 & 권한 — kubeconfig/RBAC vs IAM + IRSA**를 다룰게요! 🦥
