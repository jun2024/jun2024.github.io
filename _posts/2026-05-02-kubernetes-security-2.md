---
title: "[Kubernetes] K8s 보안 완벽 정리 (2) — TLS in Kubernetes, 인증서 생성과 Certificate API"
excerpt: "K8s 컴포넌트 간 TLS 통신 구조, openssl로 인증서 생성하기, Certificate Signing Request API로 인증서 승인까지!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Security, TLS, Certificate, CKA, DevOps]

permalink: /kubernetes/kubernetes-security-2/

toc: true
toc_sticky: true

date: 2026-05-02
last_modified_at: 2026-05-02
---

> 이전 포스트 [K8s 보안 (1)](/kubernetes/kubernetes-security-1/)에서 Security Primitives, Authentication, TLS 기초를 다뤘어요.
> 이번에는 K8s 내부의 TLS 구조와 인증서 관리를 알아볼게요!

---

## 🦥 TLS in Kubernetes — 누가 누구에게 인증서가 필요한가?

K8s 클러스터 내부의 모든 통신은 TLS로 암호화돼요. 각 컴포넌트는 **서버 인증서**와 **클라이언트 인증서**를 가지고 있어요.

### 서버 인증서 (Server Certificates)

클라이언트의 요청을 받는 쪽이 가지는 인증서예요.

| 컴포넌트 | 인증서 | 키 |
|----------|--------|-----|
| **kube-apiserver** | `apiserver.crt` | `apiserver.key` |
| **etcd server** | `etcdserver.crt` | `etcdserver.key` |
| **kubelet** | `kubelet.crt` | `kubelet.key` |

### 클라이언트 인증서 (Client Certificates)

서버에 요청하는 쪽이 가지는 인증서예요.

| 클라이언트 | 접속 대상 | 인증서 | 키 |
|-----------|----------|--------|-----|
| **admin (kubectl)** | apiserver | `admin.crt` | `admin.key` |
| **kube-scheduler** | apiserver | `scheduler.crt` | `scheduler.key` |
| **kube-controller-manager** | apiserver | `controller-manager.crt` | `controller-manager.key` |
| **kube-proxy** | apiserver | `kube-proxy.crt` | `kube-proxy.key` |
| **apiserver** | etcd | `apiserver-etcd-client.crt` | `apiserver-etcd-client.key` |
| **apiserver** | kubelet | `apiserver-kubelet-client.crt` | `apiserver-kubelet-client.key` |

### CA (Certificate Authority)

모든 인증서를 서명하는 최상위 인증 기관이에요. K8s 클러스터는 최소 **1개의 CA**가 필요해요.

| CA | 용도 | 파일 |
|----|------|------|
| **Cluster CA** | K8s 컴포넌트 인증서 서명 | `ca.crt`, `ca.key` |
| **ETCD CA** (선택) | etcd 전용 CA (분리 가능) | `etcd-ca.crt`, `etcd-ca.key` |

---

## 🦥 인증서 생성 (openssl)

### Step 1: CA 인증서 생성

```bash
# 1. CA 개인키 생성
openssl genrsa -out ca.key 2048

# 2. CA 인증서 서명 요청 (CSR)
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# 3. 자체 서명 CA 인증서 생성
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt -days 365
```

### Step 2: Admin 사용자 인증서 생성

```bash
# 1. 개인키 생성
openssl genrsa -out admin.key 2048

# 2. CSR 생성 (O=system:masters → 관리자 그룹)
openssl req -new -key admin.key \
  -subj "/CN=kube-admin/O=system:masters" -out admin.csr

# 3. CA로 서명
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out admin.crt -days 365
```

> 💡 **CKA 포인트**: `/O=system:masters`는 K8s 빌트인 관리자 그룹이에요. 이 그룹에 속한 사용자는 클러스터 전체 관리 권한을 가져요.

### Step 3: kube-apiserver 인증서 생성

apiserver는 여러 이름으로 접근될 수 있어서 **SAN (Subject Alternative Name)**을 설정해야 해요.

```bash
# openssl.cnf 파일 작성
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 192.168.1.10
EOF
```

```bash
# 키 생성
openssl genrsa -out apiserver.key 2048

# CSR 생성 (SAN 포함)
openssl req -new -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -config openssl.cnf -out apiserver.csr

# CA로 서명
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out apiserver.crt \
  -extensions v3_req -extfile openssl.cnf -days 365
```

---

## 🦥 인증서 확인 (View Certificate Details)

### 인증서 상세 정보 확인

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

주요 확인 항목은 이래요.

| 항목 | 확인 내용 | 확인 명령어 |
|------|----------|-----------|
| **Subject** | 인증서 소유자 (CN) | `openssl x509 -in cert.crt -text \| grep Subject` |
| **Issuer** | 발급한 CA | `openssl x509 -in cert.crt -text \| grep Issuer` |
| **Validity** | 유효 기간 | `openssl x509 -in cert.crt -text \| grep -A2 Validity` |
| **SAN** | 대체 이름 (DNS, IP) | `openssl x509 -in cert.crt -text \| grep -A1 "Alternative"` |

### kubeadm 클러스터의 인증서 경로

| 인증서 | 경로 |
|--------|------|
| CA | `/etc/kubernetes/pki/ca.crt` |
| apiserver | `/etc/kubernetes/pki/apiserver.crt` |
| apiserver-etcd-client | `/etc/kubernetes/pki/apiserver-etcd-client.crt` |
| apiserver-kubelet-client | `/etc/kubernetes/pki/apiserver-kubelet-client.crt` |
| etcd server | `/etc/kubernetes/pki/etcd/server.crt` |
| etcd CA | `/etc/kubernetes/pki/etcd/ca.crt` |

### 인증서 만료 확인

```bash
# 특정 인증서 만료일 확인
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate

# kubeadm으로 전체 인증서 만료 확인
kubeadm certs check-expiration
```

### 인증서 문제 트러블슈팅

인증서 관련 문제가 생기면 이 순서로 확인해요.

| 확인 순서 | 명령어 | 확인 내용 |
|----------|--------|----------|
| 1 | `kubectl get pods -n kube-system` | 컴포넌트 Pod 상태 |
| 2 | `openssl x509 -in cert.crt -text` | 인증서 Subject, Issuer, SAN |
| 3 | `openssl x509 -in cert.crt -noout -enddate` | 만료 여부 |
| 4 | `journalctl -u kubelet -f` | kubelet 로그 (인증서 오류) |
| 5 | `docker logs <etcd-container>` | etcd 컨테이너 로그 |

---

## 🦥 Certificate API — K8s로 인증서 관리

### 왜 Certificate API가 필요한가?

새 관리자가 팀에 합류할 때마다 CA 키로 직접 인증서를 서명하는 건 번거롭고 보안 위험이 있어요. K8s의 **Certificate API**를 사용하면 API를 통해 인증서를 요청하고 승인할 수 있어요.

### 인증서 발급 프로세스

| 단계 | 수행자 | 동작 |
|------|--------|------|
| 1 | 새 사용자 | 개인키 생성 + CSR 생성 |
| 2 | 새 사용자 | CSR을 K8s CertificateSigningRequest 리소스로 제출 |
| 3 | 관리자 | CSR 요청 확인 후 승인 (kubectl certificate approve) |
| 4 | K8s | CA로 자동 서명, 인증서 발급 |
| 5 | 새 사용자 | 발급된 인증서 다운로드 |

### 실습: 새 사용자 인증서 발급

**Step 1: 새 사용자가 키와 CSR 생성**

```bash
# 개인키 생성
openssl genrsa -out jane.key 2048

# CSR 생성
openssl req -new -key jane.key -subj "/CN=jane/O=developers" -out jane.csr
```

**Step 2: CertificateSigningRequest 리소스 생성**

CSR 파일을 Base64 인코딩해서 YAML에 넣어요.

```bash
cat jane.csr | base64 | tr -d '\n'
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: <BASE64_ENCODED_CSR>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

```bash
kubectl apply -f jane-csr.yaml
```

**Step 3: 관리자가 CSR 승인**

```bash
# CSR 목록 확인
kubectl get csr

# CSR 승인
kubectl certificate approve jane

# 또는 거부
kubectl certificate deny jane
```

**Step 4: 인증서 추출**

```bash
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 --decode > jane.crt
```

### Certificate API 관련 명령어

| 명령어 | 설명 |
|--------|------|
| `kubectl get csr` | CSR 목록 확인 |
| `kubectl certificate approve <name>` | CSR 승인 |
| `kubectl certificate deny <name>` | CSR 거부 |
| `kubectl delete csr <name>` | CSR 삭제 |

> 💡 Certificate API는 내부적으로 **kube-controller-manager**가 처리해요. controller-manager 설정에 `--cluster-signing-cert-file`과 `--cluster-signing-key-file`로 CA 인증서/키 경로가 지정되어 있어야 해요.

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **TLS in K8s** | 모든 컴포넌트가 서버/클라이언트 인증서 사용, CA가 서명 |
| **인증서 생성** | `openssl genrsa` → `openssl req` → `openssl x509` |
| **apiserver SAN** | kubernetes, kubernetes.default 등 여러 DNS/IP 필수 |
| **인증서 확인** | `openssl x509 -text`, `kubeadm certs check-expiration` |
| **Certificate API** | CSR 리소스 생성 → `kubectl certificate approve`로 승인 |

다음 포스트에서는 **KubeConfig, API Groups, Authorization, RBAC**를 다뤄볼게요! 🦥
