---
title: "[Kubernetes] K8s Storage (1) — Docker Storage, CSI, Volumes 완벽 가이드"
excerpt: "컨테이너 스토리지의 기초부터 K8s Volumes까지 — Docker Storage Driver, Volume Plugin, CSI, emptyDir, hostPath를 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Storage, Docker, CSI, Volumes, CKA, DevOps]

permalink: /kubernetes/kubernetes-storage-1/

toc: true
toc_sticky: true

date: 2026-05-21
last_modified_at: 2026-05-21
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Storage 섹션을 기반으로 정리한 내용이에요.

---

## 🦥 Docker Storage 기초

K8s Storage를 이해하려면 먼저 Docker가 데이터를 어떻게 저장하는지 알아야 해요.

### Docker의 파일 시스템 구조

Docker는 **Layered Architecture**로 이미지를 빌드해요. 각 Dockerfile 명령어가 하나의 레이어를 만들어요.

```dockerfile
FROM ubuntu           # Layer 1: Base OS
RUN apt-get update    # Layer 2: Package updates
RUN pip install flask # Layer 3: Dependencies
COPY . /opt/app       # Layer 4: Source code
ENTRYPOINT flask run  # Layer 5: Entrypoint
```

| 레이어 종류 | 특징 | 수정 가능 여부 |
|------------|------|--------------|
| **Image Layer** | Dockerfile 빌드 시 생성 | 읽기 전용 (Read-Only) |
| **Container Layer** | 컨테이너 실행 시 생성 | 읽기/쓰기 (Read-Write) |

### Copy-on-Write (CoW) 메커니즘

컨테이너가 Image Layer의 파일을 수정하면 어떻게 될까요? Docker는 **Copy-on-Write** 방식을 사용해요.

| 동작 | 설명 |
|------|------|
| 파일 읽기 | Image Layer에서 직접 읽음 |
| 파일 수정 | Container Layer로 복사 후 수정 (CoW) |
| 파일 생성 | Container Layer에 직접 생성 |
| 컨테이너 삭제 시 | Container Layer의 모든 데이터 삭제 |

> 💡 컨테이너가 삭제되면 Container Layer의 데이터도 함께 사라져요. 그래서 **Volume**이 필요해요!

---

## 🦥 Docker Volumes

### Volume으로 데이터 영구 저장

```bash
# Volume 생성
docker volume create data_volume

# Volume을 마운트하여 컨테이너 실행
docker run -v data_volume:/var/lib/mysql mysql

# 호스트 경로를 직접 마운트 (Bind Mount)
docker run -v /data/mysql:/var/lib/mysql mysql

# 최신 문법 (--mount)
docker run --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql
```

| 마운트 타입 | 설명 | 경로 |
|-----------|------|------|
| **Volume Mount** | Docker가 관리하는 Volume 마운트 | `/var/lib/docker/volumes/` |
| **Bind Mount** | 호스트의 임의 경로를 마운트 | 호스트의 지정 경로 |

### Storage Driver

Docker의 레이어 관리와 CoW를 담당하는 건 **Storage Driver**예요. Volume 관리와는 별개예요.

| Storage Driver | 지원 OS | 특징 |
|---------------|---------|------|
| `overlay2` | Linux (기본) | 가장 널리 사용, 성능 우수 |
| `aufs` | Ubuntu (구버전) | 초기 Docker 기본값 |
| `devicemapper` | CentOS/RHEL | 블록 레벨 스토리지 |
| `btrfs` | Linux | Btrfs 파일 시스템 전용 |
| `zfs` | Linux | ZFS 파일 시스템 전용 |

> 💡 Storage Driver는 이미지 레이어와 컨테이너 레이어를 관리하고, Volume Driver는 Volume을 관리해요. 역할이 달라요!

---

## 🦥 Volume Driver Plugin

Volume의 생성과 관리는 **Volume Driver Plugin**이 담당해요. 기본 플러그인은 `local`이에요.

| Volume Driver | 설명 |
|--------------|------|
| `local` | 호스트 로컬 스토리지 (기본값) |
| `Azure File Storage` | Azure 클라우드 스토리지 |
| `Convoy` | 범용 Volume 관리 |
| `NetApp` | NetApp 스토리지 연동 |
| `REX-Ray` | 멀티 클라우드 스토리지 (EBS, S3, etc.) |
| `VMware vSphere Storage` | vSphere VMDK 연동 |

```bash
# 특정 Volume Driver를 지정하여 컨테이너 실행
docker run --volume-driver rexray/ebs \
  --mount src=ebs-vol,target=/var/lib/mysql \
  mysql
```

이렇게 하면 AWS EBS에 볼륨이 생성되고, 컨테이너가 삭제되어도 데이터는 클라우드에 남아있어요.

---

## 🦥 Container Storage Interface (CSI)

### CSI가 필요한 이유

예전에는 K8s 코드 안에 각 스토리지 벤더의 드라이버가 직접 포함되어 있었어요. 새 스토리지를 지원하려면 K8s 소스 코드를 수정해야 했죠.

| 구분 | In-Tree (구방식) | CSI (신방식) |
|------|-----------------|-------------|
| 드라이버 위치 | K8s 소스 코드 내부 | 별도 플러그인 |
| 새 스토리지 추가 | K8s 릴리스 필요 | 벤더가 독립 개발 |
| 유지보수 | K8s 팀 부담 | 벤더가 직접 관리 |
| 호환성 | K8s 전용 | K8s, Mesos, Cloud Foundry 등 |

### CSI 동작 방식

CSI는 **Container Orchestrator**와 **Storage Provider** 사이의 표준 인터페이스예요. RPC(Remote Procedure Call)로 통신해요.

| CSI RPC | 역할 | 설명 |
|---------|------|------|
| `CreateVolume` | 볼륨 생성 | 스토리지 프로바이더에 볼륨 생성 요청 |
| `DeleteVolume` | 볼륨 삭제 | 사용이 끝난 볼륨 제거 |
| `ControllerPublishVolume` | 노드에 연결 | 볼륨을 특정 노드에 Attach |
| `NodeStageVolume` | 마운트 준비 | 노드에서 볼륨 포맷/마운트 준비 |
| `NodePublishVolume` | Pod에 마운트 | 컨테이너가 사용할 수 있도록 마운트 |

```
PVC 생성 → K8s가 CSI Driver에 CreateVolume RPC 호출
  → Storage Provider가 볼륨 생성 → PV와 바인딩
```

> 💡 **CKA 포인트**: CSI는 K8s 전용이 아니에요. 어떤 Container Orchestrator든 CSI 표준을 구현하면 동일한 스토리지 플러그인을 사용할 수 있어요.

---

## 🦥 Kubernetes Volumes

### 왜 K8s에서 Volume이 필요할까?

Docker와 마찬가지로, Pod 내 컨테이너의 데이터는 기본적으로 일시적이에요. Pod이 삭제되면 데이터도 사라져요. Volume을 사용하면 데이터를 Pod 라이프사이클 너머로 유지할 수 있어요.

### Volume 기본 사용법

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - name: alpine
    image: alpine
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out"]
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

| 필드 | 설명 |
|------|------|
| `spec.volumes` | Pod 레벨에서 Volume 정의 |
| `spec.containers.volumeMounts` | 컨테이너에서 Volume을 어느 경로에 마운트할지 |
| `mountPath` | 컨테이너 내부 마운트 경로 |
| `name` | Volume 이름 (volumes와 volumeMounts를 연결) |

### Volume Types

K8s는 다양한 Volume 타입을 지원해요.

| Volume Type | 설명 | 데이터 수명 |
|------------|------|-----------|
| `emptyDir` | Pod 생성 시 빈 디렉토리 생성 | Pod 삭제 시 삭제 |
| `hostPath` | 노드의 파일 시스템 경로 마운트 | 노드에 유지 |
| `nfs` | NFS 서버 마운트 | 영구 |
| `awsElasticBlockStore` | AWS EBS 볼륨 | 영구 |
| `gcePersistentDisk` | GCP Persistent Disk | 영구 |
| `azureDisk` | Azure Managed Disk | 영구 |
| `configMap` | ConfigMap 데이터를 파일로 마운트 | ConfigMap 수명 |
| `secret` | Secret 데이터를 파일로 마운트 | Secret 수명 |

### emptyDir

```yaml
volumes:
- name: cache-volume
  emptyDir:
    medium: Memory    # 메모리 기반 (tmpfs), 생략하면 디스크
    sizeLimit: 500Mi  # 최대 크기 제한
```

| 특징 | 설명 |
|------|------|
| 생성 시점 | Pod이 노드에 스케줄될 때 |
| 초기 상태 | 비어있음 |
| 공유 범위 | 같은 Pod 내 모든 컨테이너 |
| 삭제 시점 | Pod이 노드에서 제거될 때 |
| 사용 사례 | 임시 캐시, 컨테이너 간 데이터 공유 |

> 💡 `emptyDir`은 Multi-Container Pod에서 Sidecar 패턴으로 로그를 공유할 때 자주 사용해요.

### hostPath

```yaml
volumes:
- name: host-volume
  hostPath:
    path: /data/app
    type: DirectoryOrCreate   # 없으면 생성
```

| type 값 | 설명 |
|---------|------|
| `Directory` | 기존 디렉토리 (없으면 에러) |
| `DirectoryOrCreate` | 없으면 자동 생성 |
| `File` | 기존 파일 (없으면 에러) |
| `FileOrCreate` | 없으면 자동 생성 |

> ⚠️ **주의**: `hostPath`는 **싱글 노드 테스트 환경**에서만 사용하세요. 멀티 노드 클러스터에서는 Pod이 다른 노드에 스케줄되면 데이터에 접근할 수 없어요.

---

## 🦥 hostPath의 한계와 해결 방향

| 문제 | 설명 |
|------|------|
| 노드 종속 | Pod이 다른 노드로 이동하면 데이터 접근 불가 |
| 수동 관리 | 개발자가 스토리지 세부 사항을 알아야 함 |
| 확장성 부족 | 대규모 환경에서 관리 어려움 |

이 문제를 해결하기 위해 K8s는 **Persistent Volume**과 **Persistent Volume Claim** 개념을 도입했어요. 다음 포스트에서 자세히 다룰게요!

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Docker Storage** | Layered Architecture, Copy-on-Write, Container Layer는 일시적 |
| **Docker Volume** | Volume Mount (Docker 관리) vs Bind Mount (호스트 경로) |
| **Storage Driver** | 이미지/컨테이너 레이어 관리 (overlay2, aufs 등) |
| **Volume Driver** | Volume 관리 (local, REX-Ray 등) |
| **CSI** | Container Orchestrator ↔ Storage Provider 표준 인터페이스 |
| **K8s Volumes** | emptyDir (임시), hostPath (노드 종속), 클라우드 볼륨 (영구) |
| **emptyDir** | Pod 삭제 시 사라짐, 컨테이너 간 데이터 공유에 적합 |
| **hostPath** | 싱글 노드에서만 사용, 멀티 노드에서는 PV/PVC 사용 |

다음 포스트에서는 **Persistent Volume, PVC, Storage Class**로 스토리지를 효율적으로 관리하는 방법을 알아볼게요! 🦥
