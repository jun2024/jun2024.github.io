---
title: "[Kubernetes] 로컬 K8s vs EKS (5) — 인증 & 권한: kubeconfig/RBAC vs IAM + IRSA"
excerpt: "K8s 인증 체계가 EKS에서 어떻게 달라질까? 인증서 기반 인증 vs IAM 인증, aws-auth ConfigMap, IRSA, EKS Pod Identity까지 비교 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, EKS, AWS, IAM, IRSA, RBAC, Security, DevOps]

permalink: /kubernetes/local-vs-eks-5/

toc: true
toc_sticky: true

date: 2026-08-04
last_modified_at: 2026-08-04
---

> 이전 포스트 [로컬 vs EKS (4)](/kubernetes/local-vs-eks-4/)에서 스토리지 차이를 다뤘어요!

---

## 🦥 인증 체계 비교 — 완전히 다른 구조

로컬 K8s와 EKS의 **가장 복잡한 차이**가 바로 인증이에요. 로컬에서는 인증서 기반이지만, EKS에서는 **IAM과 K8s RBAC이 이중으로** 동작해요.

| 단계 | 로컬 K8s | EKS |
|------|---------|-----|
| **인증 (Authentication)** | 인증서, 토큰, ServiceAccount | **IAM** → aws-auth → K8s 사용자 매핑 |
| **인가 (Authorization)** | RBAC (Role/RoleBinding) | RBAC (동일) |
| **Pod의 AWS 접근** | 해당 없음 | **IRSA / EKS Pod Identity** |

---

## 🦥 사용자 인증 — 로컬 K8s

### 인증서 기반 인증

kubeadm 클러스터에서는 **X.509 인증서**로 사용자를 인증해요.

```bash
# kubeconfig 확인
cat ~/.kube/config
```

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <CA-CERT>
    server: https://192.168.1.10:6443
  name: kubernetes
users:
- name: kubernetes-admin
  user:
    client-certificate-data: <CLIENT-CERT>    # 인증서
    client-key-data: <CLIENT-KEY>              # 개인키
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
```

새 사용자를 추가하려면 인증서를 발급하고 RBAC을 설정해야 해요.

```bash
# 사용자 인증서 생성 (CKA에서 배운 방법)
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=user/O=dev-team"
# CSR을 K8s Certificate API로 승인...
```

---

## 🦥 사용자 인증 — EKS

### IAM 기반 인증

EKS에서는 **AWS IAM**이 인증을 담당해요. kubeconfig에 인증서 대신 **AWS CLI 명령어**가 들어가요.

```yaml
# EKS kubeconfig
users:
- name: arn:aws:eks:ap-northeast-2:123456789012:cluster/my-cluster
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args:
        - eks
        - get-token
        - --cluster-name
        - my-cluster
        - --region
        - ap-northeast-2
```

### 인증 흐름

| 단계 | 동작 |
|------|------|
| 1 | `kubectl` 실행 → `aws eks get-token` 호출 |
| 2 | AWS STS에서 임시 토큰 발급 |
| 3 | 토큰을 EKS apiserver에 전달 |
| 4 | apiserver가 **aws-auth ConfigMap**에서 IAM → K8s 사용자 매핑 확인 |
| 5 | 매핑된 K8s 사용자/그룹으로 RBAC 인가 |

### aws-auth ConfigMap

aws-auth는 **IAM 엔티티를 K8s 사용자/그룹에 매핑**하는 ConfigMap이에요.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # Node Group의 IAM Role → system:nodes 그룹
    - rolearn: arn:aws:iam::123456789012:role/EKS-NodeGroup-Role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

  mapUsers: |
    # IAM User → K8s 사용자 매핑
    - userarn: arn:aws:iam::123456789012:user/admin
      username: admin
      groups:
        - system:masters          # cluster-admin 권한

    - userarn: arn:aws:iam::123456789012:user/developer
      username: developer
      groups:
        - dev-team                # 커스텀 그룹 → RBAC으로 권한 부여

  mapAccounts: |
    # AWS 계정 전체 매핑 (드물게 사용)
    - "111122223333"
```

> ⚠️ **EKS 가장 흔한 실수**: aws-auth ConfigMap을 잘못 편집하면 **클러스터 접근이 완전히 차단**될 수 있어요. 수정 전 반드시 백업하세요!

```bash
# aws-auth 백업
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml

# aws-auth 수정 (eksctl 권장)
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --arn arn:aws:iam::123456789012:user/developer \
  --username developer \
  --group dev-team
```

### EKS Access Entry (신규 방식)

aws-auth를 대체하는 **EKS Access Entry** API가 나왔어요. ConfigMap 대신 AWS API로 접근 제어를 관리해요.

```bash
# Access Entry로 사용자 추가
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/developer

# Access Policy 연결
aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789012:user/developer \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=dev
```

| 비교 | aws-auth ConfigMap | EKS Access Entry |
|------|-------------------|-----------------|
| 관리 방식 | kubectl로 ConfigMap 수정 | AWS API/Console |
| 잠금 위험 | 실수 시 접근 차단 가능 | API 레벨 관리로 안전 |
| 감사 | K8s audit log | CloudTrail |
| 권장 | 기존 클러스터 | **신규 클러스터 권장** |

---

## 🦥 RBAC — 동일하지만 연결 방식이 다름

RBAC 자체는 로컬과 EKS에서 동일해요. 차이는 **누가 어떤 K8s 사용자/그룹에 매핑되는가**예요.

```yaml
# dev-team 그룹에 dev 네임스페이스 권한 부여 (로컬/EKS 동일)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
- kind: Group
  name: dev-team           # 로컬: 인증서 O= 값, EKS: aws-auth groups
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

| 항목 | 로컬 K8s | EKS |
|------|---------|-----|
| 사용자 식별 | 인증서 CN | aws-auth username |
| 그룹 식별 | 인증서 O | aws-auth groups |
| Role/RoleBinding | 동일 | 동일 |
| ClusterRole/ClusterRoleBinding | 동일 | 동일 |

---

## 🦥 IRSA — Pod에 IAM 역할 부여

### 왜 IRSA가 필요할까?

EKS에서 Pod이 S3, DynamoDB 같은 AWS 서비스에 접근하려면 **IAM 권한**이 필요해요.

| 방법 | 보안 수준 | 설명 |
|------|----------|------|
| 노드 IAM Role | 낮음 | 노드의 모든 Pod이 같은 권한 (비추천) |
| AccessKey 환경변수 | 매우 낮음 | 키 유출 위험 (절대 비추천) |
| **IRSA** | **높음** | Pod별로 다른 IAM Role 부여 |
| **EKS Pod Identity** | **높음** | IRSA 개선판 (설정 간소화) |

### IRSA 동작 원리

```
Pod (ServiceAccount)
  → OIDC 토큰 자동 주입 (AWS_WEB_IDENTITY_TOKEN_FILE)
  → AWS STS AssumeRoleWithWebIdentity
  → 임시 자격 증명 발급
  → AWS API 호출
```

### IRSA 설정

```bash
# 1. OIDC Provider 연결 (클러스터당 한 번)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster --approve

# 2. IAM Role + ServiceAccount 생성
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace my-app \
  --name s3-reader-sa \
  --role-name EKS-S3-Reader-Role \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

```yaml
# 3. Pod에서 ServiceAccount 사용
apiVersion: v1
kind: Pod
metadata:
  name: s3-reader
  namespace: my-app
spec:
  serviceAccountName: s3-reader-sa     # IRSA가 연결된 SA
  containers:
  - name: app
    image: amazon/aws-cli
    command: ["aws", "s3", "ls"]       # IAM Role 자동 사용
```

### EKS Pod Identity (IRSA 후속)

IRSA보다 설정이 간단한 **EKS Pod Identity**가 나왔어요.

```bash
# Pod Identity Agent Add-on 설치
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name eks-pod-identity-agent

# Pod Identity Association 생성
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace my-app \
  --service-account s3-reader-sa \
  --role-arn arn:aws:iam::123456789012:role/S3-Reader-Role
```

| 비교 | IRSA | EKS Pod Identity |
|------|------|-----------------|
| OIDC Provider | 필요 | 불필요 |
| IAM Trust Policy | OIDC 엔드포인트별 설정 | 간소화 |
| 크로스 계정 | 복잡 | 간편 |
| 설정 단계 | 3단계 | 2단계 |
| 권장 | 기존 클러스터 | **신규 클러스터 권장** |

---

## 🦥 정리

| 주제 | 로컬 K8s | EKS |
|------|---------|-----|
| **사용자 인증** | X.509 인증서 | IAM → aws-auth / Access Entry |
| **kubeconfig** | 인증서 + 키 | `aws eks get-token` 명령어 |
| **사용자 매핑** | 인증서 CN/O | aws-auth ConfigMap |
| **RBAC** | 동일 | 동일 |
| **Pod → AWS 접근** | 해당 없음 | IRSA / EKS Pod Identity |
| **최소 권한 원칙** | RBAC으로 구현 | RBAC + IAM 이중 적용 |

다음 포스트에서는 **업그레이드 & 유지보수 비교**를 다룰게요! 🦥
