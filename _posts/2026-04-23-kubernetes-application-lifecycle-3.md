---
title: "[Kubernetes] K8s 애플리케이션 생명주기 관리 (3) — Multi-Container Pods & InitContainers"
excerpt: "하나의 Pod에 여러 컨테이너를 함께 배치하는 패턴(Sidecar, Ambassador, Adapter)과 초기화 전용 InitContainer까지 — CKA 핵심 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, MultiContainer, InitContainer, CKA, DevOps]

permalink: /kubernetes/kubernetes-application-lifecycle-3/

toc: true
toc_sticky: true

date: 2026-04-23
last_modified_at: 2026-04-23
---

> 이 포스트는 [KodeKloud CKA 코스](https://github.com/kodekloudhub/certified-kubernetes-administrator-course)의 Application Lifecycle Management 섹션을 기반으로 정리한 내용이에요.
> 이전 포스트 [애플리케이션 생명주기 (2)](/kubernetes/kubernetes-application-lifecycle-2/)에서 환경변수, ConfigMap, Secret을 다뤘어요!

---

## 🦥 왜 하나의 Pod에 여러 컨테이너를 넣을까?

K8s에서는 보통 하나의 Pod에 하나의 컨테이너를 실행하는 게 기본이에요. 하지만 때로는 **서로 밀접하게 연관된 컨테이너**를 같은 Pod에 배치해야 할 때가 있어요.

같은 Pod에 있는 컨테이너들은 이런 것들을 **공유**해요.

| 공유 항목 | 설명 |
|----------|------|
| **네트워크** | 같은 localhost, 같은 IP 주소, 같은 포트 범위 |
| **스토리지 (Volume)** | 같은 Volume을 마운트해서 파일 공유 가능 |
| **생명주기** | Pod이 생성/삭제되면 모든 컨테이너가 함께 생성/삭제 |
| **IPC** | 프로세스 간 통신 가능 |

> 💡 "함께 배포되고, 함께 스케일링되고, 같은 네트워크/스토리지를 공유해야 하는 컨테이너"라면 같은 Pod에 넣는 게 맞아요.

---

## 🦥 Multi-Container Pod 디자인 패턴

Multi-Container Pod에는 대표적인 3가지 디자인 패턴이 있어요. CKA 시험에서 개념 문제로 자주 출제돼요.

### 1. Sidecar 패턴

메인 애플리케이션 옆에 **보조 기능**을 수행하는 컨테이너를 배치하는 패턴이에요. 가장 많이 사용되는 패턴이에요.

**대표 사례**: 로그 수집, 파일 동기화, 프록시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # 메인 컨테이너: 웹 애플리케이션
  - name: app
    image: my-web-app:1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app

  # Sidecar: 로그 수집 에이전트
  - name: log-agent
    image: fluentd:latest
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
      readOnly: true

  volumes:
  - name: log-volume
    emptyDir: {}
```

이 예시에서는 메인 앱이 `/var/log/app`에 로그를 쓰면, Sidecar인 fluentd가 같은 Volume을 읽어서 중앙 로그 시스템으로 전송해요.

### 2. Ambassador 패턴

메인 컨테이너의 **외부 통신을 대리(Proxy)**하는 컨테이너를 배치하는 패턴이에요.

**대표 사례**: DB 프록시, API Gateway, 로컬 캐시 프록시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
spec:
  containers:
  # 메인 컨테이너: localhost:3306으로 DB 접근
  - name: app
    image: my-app:1.0
    env:
    - name: DB_HOST
      value: "localhost"
    - name: DB_PORT
      value: "3306"

  # Ambassador: 외부 DB로 프록시
  - name: db-proxy
    image: mysql-proxy:latest
    ports:
    - containerPort: 3306
    env:
    - name: MYSQL_HOST
      value: "prod-mysql.database.svc.cluster.local"
```

메인 앱은 항상 `localhost:3306`으로 DB에 접근해요. Ambassador 컨테이너가 환경(개발/스테이징/프로덕션)에 따라 실제 DB로 프록시해줘요. 메인 앱의 코드를 수정할 필요가 없다는 게 장점이에요.

### 3. Adapter 패턴

메인 컨테이너의 **출력을 표준화/변환**하는 컨테이너를 배치하는 패턴이에요.

**대표 사례**: 로그 형식 변환, 메트릭 형식 통일, 데이터 포맷 변환

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
spec:
  containers:
  # 메인 컨테이너: 자체 형식으로 메트릭 출력
  - name: app
    image: legacy-app:1.0
    volumeMounts:
    - name: metrics-volume
      mountPath: /var/metrics

  # Adapter: 메트릭을 Prometheus 형식으로 변환
  - name: metrics-adapter
    image: prometheus-adapter:latest
    ports:
    - containerPort: 9090
    volumeMounts:
    - name: metrics-volume
      mountPath: /var/metrics
      readOnly: true

  volumes:
  - name: metrics-volume
    emptyDir: {}
```

레거시 앱이 자체 형식으로 메트릭을 출력하면, Adapter 컨테이너가 이를 Prometheus가 이해하는 형식으로 변환해요.

### 3가지 패턴 비교

| 패턴 | 역할 | 예시 | 통신 방향 |
|------|------|------|----------|
| **Sidecar** | 메인 앱의 보조 기능 | 로그 수집, 파일 동기화, Git sync | 메인 → (Volume) → Sidecar |
| **Ambassador** | 외부 통신 대리/프록시 | DB 프록시, API Gateway | 메인 → Ambassador → 외부 |
| **Adapter** | 출력 형식 변환/표준화 | 메트릭 변환, 로그 포맷 통일 | 메인 → (Volume) → Adapter → 외부 |

> 💡 **CKA 팁**: 시험에서는 "이 패턴이 무엇인가?" 보다 **멀티 컨테이너 Pod을 직접 만드는 문제**가 더 많이 나와요. YAML에 `containers` 배열에 여러 컨테이너를 추가하는 방법을 확실히 연습하세요!

---

## 🦥 Multi-Container Pod 생성 실습

### 기본 멀티 컨테이너 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80

  - name: log-sidecar
    image: busybox
    command: ['sh', '-c', 'while true; do echo "$(date) - sidecar running"; sleep 10; done']
```

### 컨테이너 간 Volume 공유

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'while true; do echo "$(date) - Hello from writer" >> /shared/log.txt; sleep 5; done']
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  - name: reader
    image: busybox
    command: ['sh', '-c', 'tail -f /shared/log.txt']
    volumeMounts:
    - name: shared-data
      mountPath: /shared

  volumes:
  - name: shared-data
    emptyDir: {}
```

### 멀티 컨테이너 Pod 관리 명령어

```bash
# Pod 내 컨테이너 목록 확인
kubectl describe pod multi-container

# 특정 컨테이너 로그 확인 (반드시 -c 지정)
kubectl logs multi-container -c nginx
kubectl logs multi-container -c log-sidecar

# 특정 컨테이너에 접속
kubectl exec -it multi-container -c nginx -- /bin/bash

# 특정 컨테이너의 리소스 확인
kubectl top pod multi-container --containers
```

> 💡 **CKA 포인트**: 멀티 컨테이너 Pod에서 `kubectl logs`나 `kubectl exec`를 할 때 **반드시 `-c <container-name>`을 지정**해야 해요. 안 하면 에러가 나요!

---

## 🦥 InitContainers

### InitContainer란?

InitContainer는 메인 컨테이너가 시작되기 **전에 실행**되는 초기화 전용 컨테이너예요. DB 연결 대기, 설정 파일 다운로드, 스키마 마이그레이션 같은 사전 작업에 사용해요.

### InitContainer 특징

| 특징 | 설명 |
|------|------|
| **순차 실행** | 여러 InitContainer는 정의된 순서대로 하나씩 실행 |
| **완료 후 다음 실행** | 이전 InitContainer가 성공해야 다음이 실행 |
| **모두 완료 후 메인 시작** | 모든 InitContainer가 성공해야 메인 컨테이너 시작 |
| **실패 시 재시도** | InitContainer가 실패하면 Pod의 `restartPolicy`에 따라 재시도 |
| **일회성** | 메인 컨테이너처럼 계속 실행되지 않음, 완료되면 종료 |

### 기본 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  # InitContainer 1: DB 서비스가 준비될 때까지 대기
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nslookup mysql-service.default.svc.cluster.local; do echo "Waiting for DB..."; sleep 2; done']

  # InitContainer 2: 설정 파일 다운로드
  - name: download-config
    image: busybox
    command: ['sh', '-c', 'wget -O /config/app.conf http://config-server/app.conf']
    volumeMounts:
    - name: config-volume
      mountPath: /config

  containers:
  # 메인 컨테이너: 앱 실행
  - name: app
    image: my-app:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app
    ports:
    - containerPort: 8080

  volumes:
  - name: config-volume
    emptyDir: {}
```

실행 순서는 이래요.

| 순서 | 컨테이너 | 동작 |
|------|---------|------|
| 1 | `wait-for-db` (Init) | mysql-service DNS 확인될 때까지 대기 |
| 2 | `download-config` (Init) | 설정 파일 다운로드 후 Volume에 저장 |
| 3 | `app` (Main) | Init 완료 후 앱 시작, Volume에서 설정 읽기 |

### InitContainer 실패 시 동작

```bash
# InitContainer가 실패하면 Pod 상태가 Init:Error 또는 Init:CrashLoopBackOff
kubectl get pods
```

```
NAME        READY   STATUS     RESTARTS   AGE
init-demo   0/1     Init:0/2   0          30s     # 첫 번째 Init 실행 중
init-demo   0/1     Init:1/2   0          35s     # 두 번째 Init 실행 중
init-demo   1/1     Running    0          40s     # 모든 Init 완료, 메인 실행 중
```

```
NAME        READY   STATUS                  RESTARTS   AGE
init-demo   0/1     Init:Error              1          30s     # Init 실패
init-demo   0/1     Init:CrashLoopBackOff   3          90s     # Init 반복 실패
```

### InitContainer 디버깅

```bash
# InitContainer 로그 확인
kubectl logs init-demo -c wait-for-db

# Pod 이벤트 확인
kubectl describe pod init-demo

# InitContainer 상태 상세 확인
kubectl get pod init-demo -o jsonpath='{.status.initContainerStatuses}'
```

---

## 🦥 InitContainer vs Sidecar 비교

| 항목 | InitContainer | Sidecar Container |
|------|--------------|-------------------|
| **실행 시점** | 메인 컨테이너 시작 **전** | 메인 컨테이너와 **동시** |
| **실행 방식** | 완료 후 종료 (일회성) | 메인 컨테이너와 함께 계속 실행 |
| **용도** | 사전 조건 확인, 초기화 작업 | 보조 기능 (로그 수집, 프록시) |
| **실패 시** | 메인 컨테이너 시작 안 됨 | Pod 전체에 영향 |
| **정의 위치** | `spec.initContainers` | `spec.containers` |

---

## 🦥 실전 종합 예시: 웹앱 + 초기화 + 로그 수집

InitContainer와 Sidecar를 모두 활용하는 실전 예시예요.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-full
spec:
  # InitContainer: Git에서 정적 파일 다운로드
  initContainers:
  - name: git-clone
    image: alpine/git
    command: ['git', 'clone', 'https://github.com/example/static-files.git', '/static']
    volumeMounts:
    - name: static-files
      mountPath: /static

  containers:
  # 메인: Nginx 웹서버
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: static-files
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: log-volume
      mountPath: /var/log/nginx

  # Sidecar: 로그 수집
  - name: log-collector
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
      readOnly: true

  volumes:
  - name: static-files
    emptyDir: {}
  - name: log-volume
    emptyDir: {}
```

| 컨테이너 | 타입 | 역할 |
|----------|------|------|
| `git-clone` | InitContainer | Git 저장소에서 정적 파일을 클론 (완료 후 종료) |
| `nginx` | Main | 정적 파일을 서빙하는 웹서버 |
| `log-collector` | Sidecar | Nginx 로그를 수집해서 중앙 시스템으로 전송 |

---

## 🦥 CKA 시험 빈출 포인트 정리

| 유형 | 자주 나오는 문제 | 핵심 키워드 |
|------|----------------|-----------|
| **멀티 컨테이너 생성** | "Pod에 두 번째 컨테이너를 추가하라" | `spec.containers` 배열에 추가 |
| **Volume 공유** | "두 컨테이너가 같은 디렉토리를 공유하게 하라" | `emptyDir` + 같은 `volumeMounts.mountPath` |
| **로그 확인** | "특정 컨테이너의 로그를 확인하라" | `kubectl logs <pod> -c <container>` |
| **InitContainer 추가** | "메인 앱 시작 전에 서비스를 대기하라" | `spec.initContainers` |
| **InitContainer 디버깅** | "Pod이 Init:Error 상태인 원인을 찾아라" | `kubectl logs <pod> -c <init-container>` |

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **Multi-Container** | 같은 Pod 내 컨테이너는 네트워크, Volume, 생명주기를 공유 |
| **Sidecar** | 메인 앱의 보조 기능 (로그 수집, 파일 동기화) |
| **Ambassador** | 외부 통신을 대리하는 프록시 |
| **Adapter** | 출력을 표준 형식으로 변환 |
| **InitContainer** | 메인 시작 전 초기화 작업, 순차 실행, 일회성 |
| **`-c` 옵션** | 멀티 컨테이너 Pod에서 logs/exec 시 반드시 필요 |

이것으로 K8s 애플리케이션 생명주기 관리 시리즈를 마무리할게요! Rolling Update부터 ConfigMap/Secret, Multi-Container Pods까지 — CKA 시험과 실무 모두에서 꼭 필요한 내용을 다뤘어요. 🦥
