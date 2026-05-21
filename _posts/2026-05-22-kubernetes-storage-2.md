---
title: "[Kubernetes] K8s Storage (2) — Persistent Volume, PVC, Storage Class 완벽 가이드"
excerpt: "K8s 스토리지의 핵심! PV와 PVC로 스토리지를 분리하고, Storage Class로 동적 프로비저닝하는 방법까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Storage, PV, PVC, StorageClass, CKA, DevOps]

permalink: /kubernetes/kubernetes-storage-2/

toc: true
toc_sticky: true

date: 2026-05-22
last_modified_at: 2026-05-22
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Storage 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [Storage (1)](/kubernetes/kubernetes-storage-1/)에서 Docker Storage, CSI, K8s Volumes를 다뤘어요!

---

## 🦥 왜 PV와 PVC가 필요할까?

이전 포스트에서 Volume을 Pod YAML에 직접 정의하는 방법을 배웠어요. 하지만 이 방식에는 문제가 있어요.

| 문제 | 설명 |
|------|------|
| 역할 혼재 | 개발자가 스토리지 인프라 세부 사항을 알아야 함 |
| 재사용 불가 | 같은 스토리지를 여러 Pod에서 사용하려면 매번 정의 |
| 관리 어려움 | Pod 수가 많아지면 스토리지 설정이 분산됨 |

K8s는 이 문제를 **역할 분리**로 해결해요.

| 역할 | 담당자 | 리소스 |
|------|-------|--------|
| 스토리지 프로비저닝 | 클러스터 관리자 | **Persistent Volume (PV)** |
| 스토리지 요청 | 개발자/사용자 | **Persistent Volume Claim (PVC)** |

> 💡 관리자가 PV를 미리 만들어두면, 개발자는 원하는 크기와 접근 모드로 PVC만 요청하면 돼요. 인프라 세부 사항을 알 필요 없어요!

---

## 🦥 Persistent Volume (PV)

### PV 생성

PV는 클러스터 레벨의 스토리지 리소스예요. 네임스페이스에 속하지 않아요.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/data
```

### Access Modes

| Access Mode | 약어 | 설명 |
|------------|------|------|
| `ReadWriteOnce` | RWO | 하나의 노드에서 읽기/쓰기 마운트 |
| `ReadOnlyMany` | ROX | 여러 노드에서 읽기 전용 마운트 |
| `ReadWriteMany` | RWX | 여러 노드에서 읽기/쓰기 마운트 |

> 💡 Access Mode는 **노드** 단위예요. RWO라도 같은 노드의 여러 Pod에서 동시에 마운트할 수 있어요.

### Reclaim Policy

PVC가 삭제되었을 때 PV를 어떻게 처리할지 결정해요.

| Reclaim Policy | 설명 | 사용 사례 |
|---------------|------|----------|
| `Retain` | PV 유지, 데이터 보존 (수동 처리 필요) | 중요한 데이터 |
| `Delete` | PV와 스토리지 자동 삭제 | 임시 데이터 |
| `Recycle` | 데이터 삭제 후 PV 재사용 (deprecated) | 사용하지 마세요 |

```bash
# PV 목록 확인
kubectl get pv

# PV 상세 정보
kubectl describe pv pv-vol1
```

### 다양한 스토리지 백엔드

```yaml
# AWS EBS 사용
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    volumeID: vol-0a1b2c3d4e5f
    fsType: ext4

# NFS 사용
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.example.com
    path: /exports/data
```

---

## 🦥 Persistent Volume Claim (PVC)

### PVC 생성

PVC는 개발자가 스토리지를 **요청**하는 리소스예요. K8s가 조건에 맞는 PV를 자동으로 찾아서 바인딩해요.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### PV-PVC 바인딩 규칙

K8s는 PVC의 요청 조건에 맞는 PV를 찾아서 **1:1로 바인딩**해요.

| 매칭 기준 | 설명 |
|----------|------|
| `accessModes` | PVC가 요청한 접근 모드를 PV가 지원해야 함 |
| `capacity` | PV 용량 ≥ PVC 요청 용량 |
| `storageClassName` | 같은 Storage Class여야 함 (없으면 빈 문자열끼리) |
| `selector` | PVC에 labelSelector가 있으면 PV의 label과 매칭 |

| 바인딩 상황 | 결과 |
|-----------|------|
| 조건에 맞는 PV가 있음 | 자동 바인딩 (Bound) |
| 조건에 맞는 PV가 없음 | PVC가 Pending 상태로 대기 |
| 500Mi 요청인데 1Gi PV만 있음 | 1Gi PV에 바인딩 (더 큰 PV 사용) |
| 하나의 PV에 여러 PVC | 불가 (1:1 관계) |

```bash
# PVC 목록 확인
kubectl get pvc

# PVC 상태 확인
kubectl describe pvc my-pvc
```

### PV-PVC 상태 변화

| PV 상태 | 설명 |
|---------|------|
| `Available` | 아직 바인딩되지 않은 상태 |
| `Bound` | PVC에 바인딩됨 |
| `Released` | PVC 삭제됨, 데이터 유지 중 (Retain 정책) |
| `Failed` | 자동 회수 실패 |

> 💡 `Released` 상태의 PV는 다른 PVC에 자동으로 바인딩되지 않아요. 관리자가 직접 데이터를 정리하고 PV를 삭제/재생성해야 해요.

---

## 🦥 Pod에서 PVC 사용하기

### Pod YAML에 PVC 연결

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc       # PVC 이름 참조
```

### Deployment에서 PVC 사용

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: my-storage
      volumes:
      - name: my-storage
        persistentVolumeClaim:
          claimName: my-pvc
```

> ⚠️ **주의**: RWO PVC를 Deployment에서 사용하면, 모든 Pod이 같은 노드에 스케줄되어야 해요. 여러 노드에 분산되어야 한다면 RWX를 지원하는 스토리지가 필요해요.

### PVC 삭제 시 동작

```bash
# PVC 삭제
kubectl delete pvc my-pvc
```

| 상황 | 결과 |
|------|------|
| PVC를 사용하는 Pod이 있음 | PVC 삭제가 보류됨 (Pod 삭제 때까지 대기) |
| Pod 삭제 후 PVC 삭제 | Reclaim Policy에 따라 PV 처리 |
| Retain 정책 | PV는 `Released` 상태, 데이터 보존 |
| Delete 정책 | PV와 스토리지 자동 삭제 |

---

## 🦥 Storage Class — 동적 프로비저닝

### 정적 프로비저닝의 문제

지금까지 배운 방법은 **정적 프로비저닝(Static Provisioning)**이에요.

| 단계 | 정적 프로비저닝 |
|------|---------------|
| 1 | 관리자가 실제 스토리지 생성 (EBS, GCE Disk 등) |
| 2 | 관리자가 PV 생성 (스토리지와 연결) |
| 3 | 개발자가 PVC 생성 |
| 4 | PVC-PV 바인딩 |

매번 스토리지와 PV를 수동으로 만들어야 해서 번거로워요. **Storage Class**를 사용하면 PVC가 생성될 때 스토리지와 PV가 **자동으로** 만들어져요.

### Storage Class 생성

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd    # 스토리지 프로바이더
parameters:
  type: pd-standard                   # 프로바이더별 파라미터
  replication-type: none
```

### 다양한 Provisioner

| Provisioner | 클라우드/스토리지 | 예시 타입 |
|------------|----------------|----------|
| `kubernetes.io/gce-pd` | GCP Persistent Disk | pd-standard, pd-ssd |
| `kubernetes.io/aws-ebs` | AWS EBS | gp2, gp3, io1 |
| `kubernetes.io/azure-disk` | Azure Managed Disk | Standard_LRS, Premium_LRS |
| `kubernetes.io/no-provisioner` | 로컬 스토리지 | 동적 프로비저닝 없음 |

### PVC에서 Storage Class 사용

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: google-storage    # Storage Class 이름 지정
  resources:
    requests:
      storage: 500Mi
```

> 💡 **핵심**: Storage Class를 사용하면 PV를 직접 만들 필요가 없어요. PVC가 생성되면 Storage Class가 자동으로 스토리지와 PV를 프로비저닝해요.

### 동적 프로비저닝 흐름

| 단계 | 동적 프로비저닝 |
|------|---------------|
| 1 | 관리자가 Storage Class 생성 (한 번만) |
| 2 | 개발자가 `storageClassName`을 지정한 PVC 생성 |
| 3 | Storage Class가 자동으로 스토리지 프로비저닝 |
| 4 | PV 자동 생성 및 PVC와 바인딩 |

### 여러 Storage Class 운영

실제 환경에서는 성능과 비용에 따라 여러 Storage Class를 만들어요.

```yaml
# 표준 스토리지 (저비용)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
---
# SSD 스토리지 (고성능)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
---
# SSD + 복제 (고가용성)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-replicated
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

```bash
# Storage Class 목록 확인
kubectl get sc

# 기본 Storage Class 확인 (default 어노테이션)
kubectl get sc -o wide
```

### Volume Binding Mode

| Binding Mode | 설명 |
|-------------|------|
| `Immediate` | PVC 생성 즉시 프로비저닝 (기본값) |
| `WaitForFirstConsumer` | Pod이 스케줄될 때까지 프로비저닝 대기 |

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

> 💡 `WaitForFirstConsumer`는 Pod이 스케줄될 노드를 먼저 정한 후, 해당 노드에서 스토리지를 프로비저닝해요. 노드별 로컬 스토리지를 사용할 때 필수예요.

---

## 🦥 CKA 시험 대비 — Storage 치트시트

### PV 생성 (필수 암기)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/data
```

### PVC 생성 (필수 암기)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### Pod에 PVC 연결 (필수 암기)

```yaml
volumes:
- name: my-storage
  persistentVolumeClaim:
    claimName: my-pvc
```

### 빠른 확인 명령어

```bash
kubectl get pv                    # PV 목록
kubectl get pvc                   # PVC 목록
kubectl get sc                    # Storage Class 목록
kubectl describe pv <name>       # PV 상세 정보 (바인딩 상태 등)
kubectl describe pvc <name>      # PVC 상세 정보
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **PV** | 클러스터 레벨 스토리지, 관리자가 생성, 네임스페이스 독립 |
| **PVC** | 개발자의 스토리지 요청, PV와 1:1 바인딩 |
| **Access Modes** | RWO (단일 노드), ROX (다수 노드 읽기), RWX (다수 노드 읽기/쓰기) |
| **Reclaim Policy** | Retain (보존), Delete (삭제), Recycle (deprecated) |
| **Storage Class** | 동적 프로비저닝, PVC 생성 시 자동으로 스토리지+PV 생성 |
| **Binding Mode** | Immediate (즉시) vs WaitForFirstConsumer (Pod 스케줄 후) |
| **정적 vs 동적** | 정적: PV 수동 생성 → 동적: Storage Class가 자동 생성 |

이것으로 Storage 섹션을 마무리할게요! Docker Storage의 기초부터 K8s의 PV, PVC, Storage Class까지 — CKA 시험에서 자주 출제되는 핵심 내용을 모두 다뤘어요. 🦥
