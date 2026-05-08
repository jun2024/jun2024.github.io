---
title: "[Kubernetes] K8s 보안 완벽 정리 (1) — Security Primitives, Authentication, TLS 기초"
excerpt: "K8s 보안의 기본 원칙부터 사용자 인증 방식, TLS 암호화의 동작 원리까지 — CKA 보안 섹션의 기초를 다져보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Security, Authentication, TLS, CKA, DevOps]

permalink: /kubernetes/kubernetes-security-1/

toc: true
toc_sticky: true

date: 2026-05-07
last_modified_at: 2026-05-07
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Security 섹션을 기반으로 정리한 내용이에요.
> 보안은 내용이 많아서 4개 포스트로 나눠서 다뤄요!

---

## 🦥 Kubernetes Security Primitives

### 클러스터 보안의 2가지 핵심 질문

K8s 보안은 크게 두 가지 질문으로 나눌 수 있어요.

| 질문 | 영역 | K8s 메커니즘 |
|------|------|------------|
| **누가 접근할 수 있는가?** | Authentication (인증) | 사용자/ServiceAccount 인증 |
| **무엇을 할 수 있는가?** | Authorization (인가) | RBAC, ABAC, Node, Webhook |

### kube-apiserver가 보안의 중심

모든 K8s 작업은 **kube-apiserver**를 통해 이루어져요. kubectl 명령어, 내부 컴포넌트 통신, 외부 API 호출 모두 apiserver를 거쳐요. 그래서 apiserver의 접근 제어가 K8s 보안의 핵심이에요.

| 접근 단계 | 설명 | 예시 |
|----------|------|------|
| 1. **Authentication** | 요청자가 누구인지 확인 | 인증서, 토큰, 비밀번호 |
| 2. **Authorization** | 해당 작업을 할 권한이 있는지 확인 | RBAC Role/RoleBinding |
| 3. **Admission Control** | 요청 내용을 검증/변경 | ResourceQuota, PodSecurity |

### 호스트 레벨 보안

클러스터 보안은 노드 보안부터 시작해요.

| 보안 항목 | 권장 사항 |
|----------|----------|
| SSH 접근 | 비밀번호 인증 비활성화, SSH 키만 사용 |
| root 접근 | 직접 root 로그인 차단 |
| 네트워크 | 불필요한 포트 차단, 방화벽 설정 |
| OS 업데이트 | 정기적 보안 패치 적용 |

---

## 🦥 Authentication — 누가 접근하는가?

### K8s에 접근하는 사용자 타입

| 타입 | 설명 | 예시 | 관리 방식 |
|------|------|------|----------|
| **User** | 사람 사용자 | 관리자, 개발자 | K8s 외부에서 관리 (인증서, LDAP 등) |
| **ServiceAccount** | 프로세스/앱 | Prometheus, Jenkins, 커스텀 앱 | K8s 내부 리소스로 관리 |

> 💡 K8s는 **User 리소스가 없어요**. 사용자를 직접 생성/관리하는 API가 없고, 인증서나 외부 인증 시스템으로 관리해요. 반면 ServiceAccount는 `kubectl create serviceaccount`로 생성 가능해요.

### 인증 방식 (Authentication Mechanisms)

kube-apiserver가 지원하는 인증 방식은 여러 가지예요.

| 인증 방식 | 설명 | 프로덕션 권장 |
|----------|------|-------------|
| **Static Password File** | CSV 파일에 비밀번호 저장 | X (deprecated) |
| **Static Token File** | CSV 파일에 토큰 저장 | X (deprecated) |
| **Client Certificates** | X.509 인증서 기반 인증 | O |
| **Service Account Tokens** | ServiceAccount 전용 토큰 | O |
| **OIDC (OpenID Connect)** | 외부 인증 서버 연동 | O (대규모 환경) |
| **Webhook Token Auth** | 외부 서비스에 인증 위임 | O |

### Static Password/Token File (참고용)

현재는 deprecated되었지만 개념을 이해하는 데 도움이 돼요.

```csv
# user-details.csv (비밀번호 파일)
password123,user1,u0001,group1
password456,user2,u0002,group2
```

```
# kube-apiserver 설정
--basic-auth-file=user-details.csv    # 비밀번호 인증
--token-auth-file=user-token.csv      # 토큰 인증
```

> 💡 Static 파일 방식은 **보안에 취약**하고 K8s 1.19부터 deprecated 됐어요. CKA에서는 개념만 알면 되고, 실무에서는 **인증서 기반 인증**을 사용해요.

### Client Certificate 인증 (가장 중요!)

K8s에서 가장 많이 사용하는 인증 방식이에요. 사용자가 자신의 인증서를 apiserver에 제출하고, apiserver가 CA 인증서로 검증하는 방식이에요.

```bash
# 인증서로 API 호출
curl https://kube-apiserver:6443/api/v1/pods \
  --key admin.key \
  --cert admin.crt \
  --cacert ca.crt
```

```bash
# kubectl은 kubeconfig 파일에서 인증서 경로를 참조
kubectl get pods
# → ~/.kube/config에 설정된 client-certificate, client-key 사용
```

### ServiceAccount

Pod 내부의 애플리케이션이 K8s API에 접근할 때 사용해요.

```bash
# ServiceAccount 생성
kubectl create serviceaccount my-app-sa

# ServiceAccount 조회
kubectl get serviceaccounts
```

```yaml
# Pod에 ServiceAccount 지정
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: my-app
    image: my-app:1.0
```

모든 네임스페이스에는 **default** ServiceAccount가 자동 생성돼요. Pod에 ServiceAccount를 지정하지 않으면 default가 사용돼요.

---

## 🦥 TLS 기초 — 왜 암호화가 필요한가?

### TLS가 해결하는 문제

네트워크 통신에서 데이터가 평문으로 전송되면 중간에서 가로챌 수 있어요. TLS는 **암호화**와 **신원 확인**으로 이 문제를 해결해요.

| 위협 | TLS 해결 방식 |
|------|-------------|
| **도청 (Eavesdropping)** | 데이터 암호화 (대칭키/비대칭키) |
| **변조 (Tampering)** | 메시지 무결성 검증 |
| **위장 (Impersonation)** | 인증서로 서버/클라이언트 신원 확인 |

### 대칭키 암호화 vs 비대칭키 암호화

| 구분 | 대칭키 (Symmetric) | 비대칭키 (Asymmetric) |
|------|-------------------|---------------------|
| **키 개수** | 1개 (같은 키로 암호화/복호화) | 2개 (공개키 + 개인키) |
| **속도** | 빠름 | 느림 |
| **용도** | 데이터 암호화 | 키 교환, 디지털 서명 |
| **예시** | AES | RSA, ECDSA |

### TLS Handshake 과정 (간략)

| 단계 | 동작 | 설명 |
|------|------|------|
| 1 | Client → Server | "TLS 연결하자" (지원하는 암호화 방식 전송) |
| 2 | Server → Client | 서버 인증서 + 공개키 전송 |
| 3 | Client | 인증서를 CA로 검증, 대칭키 생성 후 서버 공개키로 암호화해서 전송 |
| 4 | Server | 자신의 개인키로 복호화해서 대칭키 획득 |
| 5 | 양방향 | 대칭키로 데이터 암호화 통신 |

> 💡 비대칭키는 최초 키 교환에만 사용하고, 실제 데이터 통신은 **대칭키**로 해요. 비대칭키는 느리니까요.

### 인증서(Certificate)의 역할

인증서는 "이 공개키의 주인이 진짜 해당 서버인지"를 보증하는 문서예요. **CA (Certificate Authority)**가 서명해서 신뢰를 부여해요.

| 구성 요소 | 설명 |
|----------|------|
| **Subject** | 인증서 소유자 (서버 이름, 조직 등) |
| **Public Key** | 소유자의 공개키 |
| **Issuer** | 인증서를 발급한 CA |
| **Validity** | 유효 기간 |
| **Signature** | CA의 디지털 서명 |

### 인증서 파일 확장자

K8s에서 자주 보게 되는 인증서 파일 확장자예요.

| 파일 | 내용 | 예시 |
|------|------|------|
| `*.crt`, `*.pem` | 인증서 (공개키 포함) | `server.crt`, `ca.pem` |
| `*.key` | 개인키 | `server.key`, `ca.key` |
| `*.csr` | 인증서 서명 요청 | `server.csr` |

> 💡 **외우는 팁**: `.crt`/`.pem` = 공개 (남에게 줘도 됨), `.key` = 비공개 (절대 노출 금지)

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Security Primitives** | Authentication(누구?) + Authorization(무엇?) + Admission Control |
| **접근 주체** | User (외부 관리) vs ServiceAccount (K8s 리소스) |
| **인증 방식** | Client Certificate가 가장 중요, Static 파일은 deprecated |
| **TLS** | 비대칭키로 키 교환 → 대칭키로 데이터 암호화 |
| **인증서** | CA가 서명, `.crt` = 공개, `.key` = 비공개 |

다음 포스트에서는 **K8s 내부의 TLS 통신과 인증서 생성/관리**를 다뤄볼게요! 🦥
