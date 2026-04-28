---
title: "[Kubernetes] K8s 클러스터 유지보수 (2) — Backup & Restore, ETCDCTL 완벽 가이드"
excerpt: "K8s 클러스터 데이터를 안전하게 백업하고 복구하는 방법 — Resource Config 백업, ETCD 스냅샷, etcdctl 사용법까지 CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Backup, Restore, ETCD, CKA, DevOps]

permalink: /kubernetes/kubernetes-cluster-maintenance-2/

toc: true
toc_sticky: true

date: 2026-04-30
last_modified_at: 2026-04-30
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Cluster Maintenance 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [클러스터 유지보수 (1)](/kubernetes/kubernetes-cluster-maintenance-1/)에서 OS Upgrades와 Cluster Upgrade를 다뤘어요!

---

## 🦥 왜 백업이 필요할까?

K8s 클러스터에는 모든 리소스 정의, 애플리케이션 설정, 클러스터 상태가 저장되어 있어요. 이 데이터가 유실되면 전체 서비스가 복구 불가능해질 수 있어요.

백업해야 할 대상은 크게 3가지예요.

| 백업 대상 | 내용 | 저장 위치 |
|----------|------|----------|
| **Resource Configuration** | Deployment, Service, ConfigMap 등 리소스 정의 | YAML 파일, Git |
| **ETCD Cluster** | 클러스터의 모든 상태 데이터 | etcd 데이터 디렉토리 |
| **Persistent Volumes** | 애플리케이션의 영구 데이터 | PV (EBS, NFS 등) |

---

## 🦥 방법 1: Resource Configuration 백업

### kubectl로 리소스 백업

가장 간단한 방법은 `kubectl get`으로 모든 리소스를 YAML로 내보내는 거예요.

```bash
# 전체 네임스페이스의 모든 리소스 백업
kubectl get all --all-namespaces -o yaml > all-resources-backup.yaml

# 특정 리소스 타입별 백업
kubectl get deployments --all-namespaces -o yaml > deployments-backup.yaml
kubectl get services --all-namespaces -o yaml > services-backup.yaml
kubectl get configmaps --all-namespaces -o yaml > configmaps-backup.yaml
kubectl get secrets --all-namespaces -o yaml > secrets-backup.yaml
```

### 한계점

| 한계 | 설명 |
|------|------|
| `kubectl get all`의 범위 | CRD, PV, ClusterRole 등 일부 리소스를 포함하지 않음 |
| 상태 정보 포함 | `status` 필드가 포함되어 복구 시 문제 가능 |
| 수동 작업 | 자동화하지 않으면 누락 위험 |

### 더 나은 방법: Velero

실무에서는 **Velero (구 Heptio Ark)**를 사용해서 리소스 백업을 자동화해요. Velero는 리소스 설정뿐 아니라 PV 스냅샷까지 함께 관리할 수 있어요.

> 💡 **가장 좋은 방법**: 모든 리소스를 **Git에 선언적으로 관리(GitOps)**하는 거예요. ArgoCD나 Flux를 사용하면 Git이 곧 백업이자 복구 소스가 돼요.

---

## 🦥 방법 2: ETCD 스냅샷 백업

### ETCD란?

ETCD는 K8s 클러스터의 **모든 상태 데이터를 저장하는 분산 키-값 저장소**예요. 노드 정보, Pod 상태, ConfigMap, Secret 등 모든 것이 ETCD에 저장돼요.

| 저장 데이터 | 예시 |
|------------|------|
| 클러스터 상태 | 노드 목록, 네임스페이스 |
| 리소스 정의 | Deployment, Service, ConfigMap |
| 민감 데이터 | Secret (Base64 인코딩) |
| 스케줄링 정보 | Pod-Node 바인딩 |
| RBAC | Role, RoleBinding, ServiceAccount |

> 💡 ETCD를 백업하면 **클러스터의 모든 것**을 복구할 수 있어요. 그래서 ETCD 백업이 가장 중요해요.

### ETCD 정보 확인

```bash
# ETCD Pod 확인
kubectl get pods -n kube-system | grep etcd

# ETCD Pod의 상세 정보 확인 (인증서 경로, 엔드포인트 등)
kubectl describe pod etcd-controlplane -n kube-system
```

ETCD의 주요 설정값은 이래요.

| 설정 | 설명 | 기본 경로 |
|------|------|----------|
| `--listen-client-urls` | 클라이언트 접속 엔드포인트 | `https://127.0.0.1:2379` |
| `--cert-file` | ETCD 서버 인증서 | `/etc/kubernetes/pki/etcd/server.crt` |
| `--key-file` | ETCD 서버 키 | `/etc/kubernetes/pki/etcd/server.key` |
| `--trusted-ca-file` | CA 인증서 | `/etc/kubernetes/pki/etcd/ca.crt` |
| `--data-dir` | 데이터 저장 디렉토리 | `/var/lib/etcd` |

---

## 🦥 ETCDCTL 사용법

`etcdctl`은 ETCD를 관리하는 CLI 도구예요. CKA 시험에서 매우 자주 출제돼요.

### API 버전 설정

```bash
# etcdctl은 API v2가 기본이지만, K8s는 v3를 사용
export ETCDCTL_API=3
```

> 💡 **CKA 필수**: 시험에서 `ETCDCTL_API=3`을 설정하지 않으면 명령어가 동작하지 않아요. 반드시 먼저 설정하세요!

### ETCD 상태 확인

```bash
# ETCD 클러스터 상태 확인
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# ETCD 멤버 목록
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

### 인증서 경로 확인 방법

인증서 경로를 모를 때는 ETCD Pod의 명령어 인자에서 확인할 수 있어요.

```bash
# 방법 1: describe로 확인
kubectl describe pod etcd-controlplane -n kube-system | grep -E "cert|key|ca"

# 방법 2: YAML에서 확인
kubectl get pod etcd-controlplane -n kube-system -o yaml | grep -E "cert|key|ca"

# 방법 3: etcd manifest 파일 확인
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "cert|key|ca"
```

---

## 🦥 ETCD 스냅샷 백업

### 스냅샷 생성

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

```
Snapshot saved at /opt/etcd-backup.db
```

### 스냅샷 상태 확인

```bash
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-table
```

```
+---------+----------+------------+------------+
|  HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+---------+----------+------------+------------+
| 3c9f8a2 |    12847 |       1035 |     5.1 MB |
+---------+----------+------------+------------+
```

---

## 🦥 ETCD 스냅샷 복구

ETCD 복구는 CKA 시험에서 **가장 자주 출제되는 주제** 중 하나예요. 단계를 정확히 기억해야 해요.

### Step 1: 스냅샷에서 데이터 복구

```bash
ETCDCTL_API=3 etcdctl \
  snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup
```

이 명령어는 새로운 데이터 디렉토리(`/var/lib/etcd-from-backup`)에 스냅샷 데이터를 복구해요. 기존 데이터 디렉토리를 덮어쓰지 않아요.

### Step 2: ETCD가 새 데이터 디렉토리를 사용하도록 변경

**kubeadm 클러스터** (Static Pod으로 ETCD 실행)인 경우, ETCD manifest 파일을 수정해요.

```bash
# etcd manifest 수정
vi /etc/kubernetes/manifests/etcd.yaml
```

`etcd.yaml`에서 `--data-dir`과 해당 Volume 경로를 변경해요.

```yaml
# 변경 전
spec:
  containers:
  - command:
    - etcd
    - --data-dir=/var/lib/etcd
    # ...
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
  volumes:
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data

# 변경 후
spec:
  containers:
  - command:
    - etcd
    - --data-dir=/var/lib/etcd-from-backup     # 변경
    # ...
    volumeMounts:
    - mountPath: /var/lib/etcd-from-backup      # 변경
      name: etcd-data
  volumes:
  - hostPath:
      path: /var/lib/etcd-from-backup           # 변경
      type: DirectoryOrCreate
    name: etcd-data
```

### Step 3: ETCD Pod 자동 재시작 대기

Static Pod의 manifest를 수정하면 kubelet이 자동으로 ETCD Pod을 재시작해요. 잠시 기다리면 클러스터가 복구돼요.

```bash
# ETCD Pod 상태 확인 (잠시 기다림)
kubectl get pods -n kube-system | grep etcd

# 클러스터 데이터 복구 확인
kubectl get deployments
kubectl get services
```

### 복구 전체 프로세스 요약

| 단계 | 명령어/동작 | 설명 |
|------|-----------|------|
| 1 | `etcdctl snapshot restore` | 스냅샷 → 새 데이터 디렉토리 |
| 2 | `vi /etc/kubernetes/manifests/etcd.yaml` | `--data-dir` 경로를 새 디렉토리로 변경 |
| 3 | hostPath volume 경로도 변경 | Volume 마운트 경로 일치시키기 |
| 4 | 대기 | kubelet이 Static Pod 자동 재시작 |
| 5 | `kubectl get pods -n kube-system` | ETCD Pod 정상 확인 |

> 💡 **CKA 핵심**: 복구 시 가장 많이 실수하는 부분은 `--data-dir`만 변경하고 **Volume의 hostPath를 변경하지 않는 것**이에요. 두 곳 모두 새 경로로 맞춰야 해요!

---

## 🦥 Stacked ETCD vs External ETCD

ETCD 배포 방식에 따라 백업/복구 방법이 약간 달라요.

| 항목 | Stacked ETCD | External ETCD |
|------|-------------|---------------|
| **ETCD 위치** | Control Plane 노드 내부 | 별도 서버 |
| **배포 방식** | Static Pod | systemd 서비스 |
| **설정 파일** | `/etc/kubernetes/manifests/etcd.yaml` | `/etc/systemd/system/etcd.service` |
| **일반적 환경** | kubeadm 기본 설정 | 대규모 프로덕션 |
| **복구 시** | manifest 파일 수정 | systemd 서비스 재시작 |

### External ETCD 복구 시

```bash
# 1. 스냅샷 복구
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup

# 2. ETCD 서비스 설정 변경
sudo vi /etc/systemd/system/etcd.service
# --data-dir=/var/lib/etcd-from-backup 으로 변경

# 3. 서비스 재시작
sudo systemctl daemon-reload
sudo systemctl restart etcd

# 4. kube-apiserver 재시작 (필요 시)
sudo systemctl restart kube-apiserver
```

---

## 🦥 CKA 시험 대비 — 빠른 백업/복구 치트시트

### 백업 (무조건 외우기)

```bash
export ETCDCTL_API=3

etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 복구 (무조건 외우기)

```bash
# 1. 스냅샷 복구
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup

# 2. etcd.yaml 수정 (3곳 변경)
#    - --data-dir 값
#    - volumeMounts.mountPath 값
#    - volumes.hostPath.path 값
vi /etc/kubernetes/manifests/etcd.yaml

# 3. 복구 확인
kubectl get pods -n kube-system | grep etcd
```

### 인증서 경로 빠르게 찾기

```bash
# etcd manifest에서 한 번에 확인
grep -E "cert|key|ca|data-dir" /etc/kubernetes/manifests/etcd.yaml
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **백업 대상** | Resource Config, ETCD, Persistent Volumes |
| **Resource 백업** | `kubectl get all -A -o yaml`, 실무에서는 GitOps 또는 Velero |
| **ETCD 백업** | `etcdctl snapshot save`, 인증서 3개 필수 (ca, cert, key) |
| **ETCD 복구** | `snapshot restore` → `--data-dir` 변경 → Volume 경로 변경 |
| **ETCDCTL_API=3** | 반드시 설정! 안 하면 명령어 동작 안 함 |
| **Stacked vs External** | kubeadm = Static Pod (manifest), External = systemd 서비스 |

이것으로 클러스터 유지보수 시리즈를 마무리할게요! OS Upgrades, Cluster Upgrade, 그리고 ETCD Backup & Restore까지 — CKA 시험에서 가장 자주 출제되는 내용을 다뤘어요. 🦥
