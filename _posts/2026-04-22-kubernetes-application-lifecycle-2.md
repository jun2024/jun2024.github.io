---
title: "[Kubernetes] K8s 애플리케이션 생명주기 관리 (2) — 환경변수, ConfigMap, Secret"
excerpt: "K8s에서 애플리케이션 설정을 관리하는 3가지 방법 — 환경변수 직접 주입, ConfigMap으로 설정 분리, Secret으로 민감 데이터 관리까지 완벽 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, ConfigMap, Secret, CKA, DevOps]

permalink: /kubernetes/kubernetes-application-lifecycle-2/

toc: true
toc_sticky: true

date: 2026-04-22
last_modified_at: 2026-04-22
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Application Lifecycle Management 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [애플리케이션 생명주기 (1)](/kubernetes/kubernetes-application-lifecycle-1/)에서 Rolling Update와 Commands/Arguments를 다뤘어요!

---

## 🦥 왜 설정을 분리해야 할까?

애플리케이션 코드에 DB 호스트, API 키, 포트 번호 같은 설정값을 하드코딩하면 환경(개발/스테이징/프로덕션)마다 이미지를 다시 빌드해야 해요. K8s에서는 설정을 **코드와 분리**해서 관리할 수 있어요.

| 방법 | 용도 | 민감 데이터 |
|------|------|-----------|
| **환경변수 직접 주입** | 간단한 설정값 | X |
| **ConfigMap** | 설정 파일/값을 중앙 관리 | X |
| **Secret** | 비밀번호, API 키, 인증서 | O (Base64 인코딩) |

---

## 🦥 환경변수 (Environment Variables)

### 기본 환경변수 설정

Pod에 환경변수를 직접 주입하는 가장 간단한 방법이에요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    env:
    - name: APP_COLOR
      value: "blue"
    - name: APP_MODE
      value: "production"
    - name: DB_HOST
      value: "mysql-service"
    - name: DB_PORT
      value: "3306"
```

### 환경변수 주입 방식 3가지

| 방식 | 설정 | 사용 시기 |
|------|------|----------|
| **직접 지정** | `env.value` | 간단한 값, 개수가 적을 때 |
| **ConfigMap에서 가져오기** | `env.valueFrom.configMapKeyRef` | 여러 Pod이 같은 설정을 공유할 때 |
| **Secret에서 가져오기** | `env.valueFrom.secretKeyRef` | 비밀번호, 키 등 민감한 데이터 |

```yaml
env:
# 방식 1: 직접 지정
- name: APP_COLOR
  value: "blue"

# 방식 2: ConfigMap에서 가져오기
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database_host

# 방식 3: Secret에서 가져오기
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: db_password
```

---

## 🦥 ConfigMap

ConfigMap은 설정 데이터를 키-값 쌍으로 저장하는 K8s 리소스예요. 환경변수, 설정 파일, 커맨드라인 인자 등을 Pod에서 분리해서 관리할 수 있어요.

### ConfigMap 생성

**방법 1: 명령형 (Imperative)**

```bash
# 키-값 직접 지정
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=production

# 파일에서 생성
kubectl create configmap app-config \
  --from-file=app.properties

# 디렉토리에서 생성 (디렉토리 내 모든 파일)
kubectl create configmap app-config \
  --from-file=config-dir/
```

**방법 2: 선언형 (Declarative)**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: production
  DB_HOST: mysql-service
  DB_PORT: "3306"
```

```bash
kubectl apply -f configmap.yaml
```

### ConfigMap 조회

```bash
# ConfigMap 목록
kubectl get configmaps   # 또는 kubectl get cm

# ConfigMap 상세 확인
kubectl describe configmap app-config

# YAML로 확인
kubectl get configmap app-config -o yaml
```

### ConfigMap을 Pod에 주입하기

**방법 1: 개별 키를 환경변수로 주입**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    env:
    - name: APP_COLOR
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_COLOR
```

**방법 2: ConfigMap 전체를 환경변수로 주입**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    envFrom:
    - configMapRef:
        name: app-config
```

> 💡 `envFrom`을 사용하면 ConfigMap의 **모든 키-값**이 환경변수로 자동 주입돼요. 키 이름이 그대로 환경변수 이름이 되기 때문에, 키 이름을 환경변수 규칙에 맞게 작성해야 해요.

**방법 3: Volume으로 마운트 (설정 파일)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

Volume으로 마운트하면 ConfigMap의 각 키가 파일 이름이 되고, 값이 파일 내용이 돼요.

```bash
# 마운트된 파일 확인
ls /etc/config/
# APP_COLOR  APP_MODE  DB_HOST  DB_PORT

cat /etc/config/APP_COLOR
# blue
```

### ConfigMap 주입 방식 비교

| 방식 | 사용법 | 장점 |
|------|--------|------|
| **env (개별 키)** | `valueFrom.configMapKeyRef` | 필요한 키만 선택적으로 주입 |
| **envFrom (전체)** | `envFrom.configMapRef` | ConfigMap 전체를 한 번에 주입, 간편 |
| **Volume 마운트** | `volumes.configMap` | 설정 파일로 마운트, **업데이트 시 자동 반영** |

> 💡 **중요**: 환경변수로 주입한 ConfigMap은 변경해도 Pod을 재시작해야 반영돼요. 하지만 Volume으로 마운트한 경우에는 ConfigMap 변경이 **자동으로 반영**돼요 (약간의 지연 있음).

---

## 🦥 Secret

Secret은 비밀번호, OAuth 토큰, SSH 키 같은 **민감한 데이터**를 저장하는 K8s 리소스예요. ConfigMap과 구조는 비슷하지만, 데이터가 Base64로 인코딩되어 저장돼요.

### Secret 생성

**방법 1: 명령형 (Imperative)**

```bash
kubectl create secret generic app-secret \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_USER=root \
  --from-literal=DB_PASSWORD=p@ssw0rd
```

**방법 2: 선언형 (Declarative)**

Secret의 data 필드에는 **Base64 인코딩된 값**을 넣어야 해요.

```bash
# Base64 인코딩
echo -n 'mysql' | base64          # bXlzcWw=
echo -n 'root' | base64           # cm9vdA==
echo -n 'p@ssw0rd' | base64      # cEBzc3cwcmQ=
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_HOST: bXlzcWw=
  DB_USER: cm9vdA==
  DB_PASSWORD: cEBzc3cwcmQ=
```

> 💡 `stringData` 필드를 사용하면 Base64 인코딩 없이 평문으로 작성할 수 있어요. K8s가 자동으로 인코딩해줘요.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
stringData:
  DB_HOST: mysql
  DB_USER: root
  DB_PASSWORD: p@ssw0rd
```

### Secret 조회

```bash
# Secret 목록
kubectl get secrets

# Secret 상세 (값은 숨김 처리됨)
kubectl describe secret app-secret

# Secret 값 확인 (Base64 인코딩 상태)
kubectl get secret app-secret -o yaml

# Base64 디코딩
echo -n 'cEBzc3cwcmQ=' | base64 --decode
# p@ssw0rd
```

### Secret을 Pod에 주입하기

**방법 1: 개별 키를 환경변수로 주입**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD
```

**방법 2: Secret 전체를 환경변수로 주입**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    envFrom:
    - secretRef:
        name: app-secret
```

**방법 3: Volume으로 마운트**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

```bash
# 마운트된 Secret 확인
ls /etc/secrets/
# DB_HOST  DB_USER  DB_PASSWORD

cat /etc/secrets/DB_PASSWORD
# p@ssw0rd   (자동으로 디코딩됨)
```

### Secret 보안 주의사항

Secret은 기본적으로 **암호화되지 않고 Base64 인코딩만** 되어 있어요. 실무에서는 추가 보안 조치가 필요해요.

| 보안 위협 | 대응 방안 |
|----------|----------|
| etcd에 평문 저장 | **EncryptionConfiguration**으로 etcd 암호화 활성화 |
| RBAC 미설정 시 누구나 조회 가능 | **RBAC**으로 Secret 접근 권한 제한 |
| YAML 파일에 Secret 노출 | **Sealed Secrets** 또는 **External Secrets Operator** 사용 |
| Git에 Secret 커밋 | `.gitignore` 설정, Git secret scanning 활성화 |

> 💡 **CKA 포인트**: Base64는 인코딩이지 암호화가 아니에요! `echo -n 'cEBzc3cwcmQ=' | base64 --decode`로 누구나 디코딩할 수 있어요. etcd 암호화 설정은 CKA 시험에서도 출제될 수 있어요.

### etcd에서 Secret 암호화 활성화

```yaml
# /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <BASE64_ENCODED_32_BYTE_KEY>
    - identity: {}
```

kube-apiserver에 `--encryption-provider-config` 플래그를 추가하면 돼요.

---

## 🦥 ConfigMap vs Secret 비교

| 항목 | ConfigMap | Secret |
|------|-----------|--------|
| **용도** | 일반 설정 데이터 | 민감한 데이터 |
| **저장 방식** | 평문 | Base64 인코딩 |
| **크기 제한** | 1MiB | 1MiB |
| **환경변수 주입** | `configMapRef` / `configMapKeyRef` | `secretRef` / `secretKeyRef` |
| **Volume 마운트** | `volumes.configMap` | `volumes.secret` |
| **etcd 암호화** | 기본 X | 별도 설정 필요 (EncryptionConfig) |
| **메모리 저장** | X | O (tmpfs, 디스크에 안 씀) |

> 💡 Secret을 Volume으로 마운트하면 **tmpfs (메모리 기반 파일시스템)**에 저장돼요. 노드의 디스크에는 기록되지 않아서 더 안전해요.

---

## 🦥 실전 예시: 데이터베이스 연결 설정

실무에서 자주 사용하는 DB 연결 설정 예시를 ConfigMap과 Secret을 조합해서 구성해볼게요.

```yaml
# configmap.yaml — 일반 설정
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: "mysql-service.database.svc.cluster.local"
  DB_PORT: "3306"
  DB_NAME: "myapp"
---
# secret.yaml — 민감 설정
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
stringData:
  DB_USER: admin
  DB_PASSWORD: "S3cur3P@ss!"
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: webapp:2.0
        envFrom:
        - configMapRef:
            name: db-config
        - secretRef:
            name: db-secret
        ports:
        - containerPort: 8080
```

이렇게 하면 Pod 안에서 `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` 환경변수가 모두 사용 가능해요.

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **환경변수** | `env.value`로 직접 주입, 가장 간단 |
| **ConfigMap** | 일반 설정을 키-값으로 관리, `envFrom`으로 전체 주입 가능 |
| **Secret** | 민감 데이터를 Base64 인코딩으로 저장, etcd 암호화 권장 |
| **Volume 마운트** | 설정 파일로 마운트 가능, ConfigMap은 변경 시 자동 반영 |
| **보안** | Secret ≠ 암호화, EncryptionConfig + RBAC 필수 |

다음 포스트에서는 **Multi-Container Pods와 InitContainers**를 다뤄볼게요! 🦥
