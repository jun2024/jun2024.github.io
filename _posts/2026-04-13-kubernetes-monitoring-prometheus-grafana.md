---
title: "[Kubernetes] K8s 모니터링 실무 — Prometheus + Grafana 완벽 구성 가이드"
excerpt: "Prometheus로 메트릭을 수집하고, Grafana로 시각화하는 K8s 모니터링 스택을 처음부터 구성해보자! Helm 설치부터 알림 설정까지 실무 운영 노하우를 담았어요."

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Prometheus, Grafana, Monitoring, DevOps]

permalink: /kubernetes/kubernetes-monitoring-prometheus-grafana/

toc: true
toc_sticky: true

date: 2026-04-13
last_modified_at: 2026-04-13
---

> 이전 포스트 [K8s 모니터링 & 로깅 기본 개념](/kubernetes/kubernetes-monitoring-logging-basics/)에서 Metrics Server와 kubectl 기반 모니터링을 다뤘어요.
> 이번에는 실무에서 가장 많이 사용하는 **Prometheus + Grafana** 스택을 구성해볼게요!

---

## 🦥 왜 Prometheus + Grafana인가?

Metrics Server만으로는 실무 운영이 어려워요. 과거 데이터 조회, 알림, 대시보드가 없으니까요. 그래서 대부분의 프로덕션 환경에서는 **Prometheus + Grafana** 조합을 사용해요.

| 기능 | Metrics Server | Prometheus + Grafana |
|------|---------------|---------------------|
| 실시간 메트릭 | O | O |
| 과거 데이터 조회 | X | O (시계열 DB) |
| 알림 (Alerting) | X | O (Alertmanager) |
| 대시보드 | X | O (Grafana) |
| 커스텀 메트릭 | X | O (Exporter) |
| PromQL 쿼리 | X | O |

---

## 🦥 Prometheus 아키텍처

Prometheus의 전체 아키텍처를 이해하면 구성이 훨씬 쉬워져요.

```
┌──────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                 │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │  App Pod     │  │  App Pod     │  │ Node Exporter│ │
│  │  :8080/metrics│ │  :8080/metrics│ │  (DaemonSet) │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬───────┘ │
│         └────────────────┼─────────────────┘         │
│                    Pull (Scrape)                      │
│                          │                           │
│              ┌───────────▼──────────┐                │
│              │    Prometheus Server  │                │
│              │  ┌────────────────┐  │                │
│              │  │  TSDB (시계열DB)│  │                │
│              │  └────────────────┘  │                │
│              │  ┌────────────────┐  │                │
│              │  │  PromQL Engine  │  │                │
│              │  └────────────────┘  │                │
│              └──────────┬──────────┘                │
│                    │         │                        │
│          ┌─────────▼──┐  ┌──▼───────────┐           │
│          │   Grafana   │  │ Alertmanager │           │
│          │ (시각화)    │  │  (알림 전송)  │           │
│          └────────────┘  └──────────────┘           │
└──────────────────────────────────────────────────────┘
```

### 핵심 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| **Prometheus Server** | 메트릭 수집(Pull), 저장(TSDB), 쿼리(PromQL) |
| **Exporter** | 다양한 시스템의 메트릭을 Prometheus 형식으로 변환 |
| **Alertmanager** | 알림 규칙에 따라 Slack, Email, PagerDuty 등으로 알림 전송 |
| **Grafana** | Prometheus 데이터를 시각화하는 대시보드 |
| **ServiceMonitor/PodMonitor** | K8s CRD로 스크랩 대상을 선언적으로 관리 |

### Pull 기반 메트릭 수집

Prometheus의 가장 큰 특징은 **Pull 방식**이에요. 애플리케이션이 `/metrics` 엔드포인트를 노출하면, Prometheus가 주기적으로 가져가는 구조예요.

```
Push 방식 (Datadog 등):  App → 모니터링 서버
Pull 방식 (Prometheus):  모니터링 서버 → App의 /metrics 엔드포인트
```

Pull 방식의 장점은 모니터링 대상이 죽어도 Prometheus가 감지할 수 있고, 중앙에서 수집 주기를 제어할 수 있다는 거예요.

---

## 🦥 kube-prometheus-stack 설치 (Helm)

실무에서 가장 많이 사용하는 방법은 `kube-prometheus-stack` Helm 차트를 이용한 설치예요. Prometheus, Grafana, Alertmanager, Node Exporter, kube-state-metrics를 한 번에 설치할 수 있어요.

### 사전 준비

```bash
# Helm 설치 확인
helm version

# prometheus-community 레포 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 설치

```bash
# monitoring 네임스페이스 생성 및 설치
kubectl create namespace monitoring

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=mySecurePassword \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

### 설치 확인

```bash
kubectl get pods -n monitoring
```

```
NAME                                                     READY   STATUS    RESTARTS
kube-prometheus-stack-grafana-7f8c9b4d5c-abc12           3/3     Running   0
kube-prometheus-stack-kube-state-metrics-5d8f7b9c4-xyz   1/1     Running   0
kube-prometheus-stack-operator-6c9f8d7e5-def34           1/1     Running   0
kube-prometheus-stack-prometheus-node-exporter-abcde      1/1     Running   0
alertmanager-kube-prometheus-stack-alertmanager-0         2/2     Running   0
prometheus-kube-prometheus-stack-prometheus-0             2/2     Running   0
```

### 주요 설치 옵션

| 옵션 | 설명 | 권장값 |
|------|------|--------|
| `prometheus.prometheusSpec.retention` | 데이터 보존 기간 | 15d ~ 30d |
| `prometheus.prometheusSpec.storageSpec` | PV 스토리지 설정 | SSD 50Gi 이상 |
| `grafana.adminPassword` | Grafana 관리자 비밀번호 | 강력한 비밀번호 |
| `grafana.persistence.enabled` | Grafana 대시보드 영구 저장 | true |
| `alertmanager.alertmanagerSpec.retention` | 알림 데이터 보존 | 120h |

---

## 🦥 Grafana 대시보드 접근 및 구성

### Grafana 접근

```bash
# Port-forward로 로컬 접근
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring

# 브라우저에서 http://localhost:3000 접속
# ID: admin / PW: 설치 시 설정한 비밀번호
```

프로덕션 환경에서는 Ingress를 설정해서 도메인으로 접근하는 게 일반적이에요.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-grafana
            port:
              number: 80
```

### 추천 대시보드

kube-prometheus-stack을 설치하면 기본 대시보드가 포함되어 있지만, 커뮤니티 대시보드도 추가하면 더 유용해요.

| 대시보드 ID | 이름 | 용도 |
|------------|------|------|
| **315** | Kubernetes Cluster Monitoring | 클러스터 전체 개요 |
| **6417** | Kubernetes Cluster (Prometheus) | 노드/네임스페이스별 리소스 |
| **13770** | K8s Node Exporter | 노드 상세 메트릭 |
| **7249** | Kubernetes Cluster Overview | 워크로드별 리소스 사용량 |

Grafana에서 **+ → Import → Dashboard ID 입력**으로 간단하게 추가할 수 있어요.

---

## 🦥 PromQL 기본 쿼리

Prometheus의 쿼리 언어인 PromQL을 알면 원하는 메트릭을 정확하게 조회할 수 있어요.

### 기본 문법

```promql
# 메트릭 이름으로 조회
container_cpu_usage_seconds_total

# 레이블 필터링
container_cpu_usage_seconds_total{namespace="default"}

# 여러 조건
container_memory_working_set_bytes{namespace="default", pod=~"webapp.*"}
```

### 실무에서 자주 쓰는 쿼리

**노드 CPU 사용률 (%)**
```promql
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**노드 메모리 사용률 (%)**
```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

**Pod CPU 사용량 (네임스페이스별)**
```promql
sum(rate(container_cpu_usage_seconds_total{namespace!=""}[5m])) by (namespace)
```

**Pod 메모리 사용량 (Top 10)**
```promql
topk(10, sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace))
```

**Pod 재시작 횟수**
```promql
sum(kube_pod_container_status_restarts_total) by (pod, namespace)
```

**디스크 사용률 (노드별)**
```promql
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

### PromQL 주요 함수

| 함수 | 설명 | 예시 |
|------|------|------|
| `rate()` | 초당 증가율 (counter용) | `rate(http_requests_total[5m])` |
| `irate()` | 순간 증가율 (더 민감) | `irate(cpu_usage[5m])` |
| `sum()` | 합계 | `sum by(ns)(metric)` |
| `avg()` | 평균 | `avg by(node)(metric)` |
| `topk()` | 상위 N개 | `topk(5, metric)` |
| `histogram_quantile()` | 분위수 계산 | `histogram_quantile(0.95, rate(http_duration_bucket[5m]))` |

---

## 🦥 ServiceMonitor로 커스텀 메트릭 수집

kube-prometheus-stack의 Prometheus Operator를 사용하면 `ServiceMonitor` CRD로 스크랩 대상을 선언적으로 관리할 수 있어요.

### 애플리케이션에서 메트릭 노출

먼저 애플리케이션이 `/metrics` 엔드포인트를 노출해야 해요. Python (Flask) 예시를 볼게요.

```python
# app.py
from flask import Flask
from prometheus_client import Counter, Histogram, generate_latest

app = Flask(__name__)

REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency', ['endpoint'])

@app.route('/')
def index():
    REQUEST_COUNT.labels(method='GET', endpoint='/', status='200').inc()
    return 'Hello World'

@app.route('/metrics')
def metrics():
    return generate_latest()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Service 및 ServiceMonitor 생성

```yaml
# app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
  - name: http-metrics
    port: 8080
    targetPort: 8080
---
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack    # Helm release 이름과 매칭 필요
spec:
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: http-metrics
    interval: 30s
    path: /metrics
```

```bash
kubectl apply -f app-service.yaml
kubectl apply -f servicemonitor.yaml

# Prometheus Target에서 수집 확인
# Prometheus UI → Status → Targets 에서 my-app이 UP 상태인지 확인
```

> 💡 ServiceMonitor의 `labels`에 Helm release 이름(`release: kube-prometheus-stack`)을 매칭해야 Prometheus가 인식해요. 이걸 빠뜨리면 타겟에 안 잡혀서 메트릭 수집이 안 돼요!

---

## 🦥 Alertmanager 알림 설정

모니터링의 핵심은 문제 발생 시 **자동으로 알림을 받는 것**이에요. Alertmanager를 통해 Slack, Email 등으로 알림을 보낼 수 있어요.

### 알림 규칙 생성 (PrometheusRule)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: pod-alerts
    rules:
    # Pod 재시작 알림
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }}이 반복적으로 재시작되고 있어요"
        description: "네임스페이스 {{ $labels.namespace }}의 Pod {{ $labels.pod }}이 지난 15분간 재시작되었어요."

    # 높은 CPU 사용률 알림
    - alert: HighCPUUsage
      expr: |
        (sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
        / sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod, namespace)) > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }}의 CPU 사용률이 90%를 초과했어요"

    # 높은 메모리 사용률 알림
    - alert: HighMemoryUsage
      expr: |
        (sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)
        / sum(kube_pod_container_resource_limits{resource="memory"}) by (pod, namespace)) > 0.85
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }}의 메모리 사용률이 85%를 초과했어요 — OOM 위험!"
```

### Slack 알림 연동

Helm values 파일에 Alertmanager 설정을 추가해요.

```yaml
# alertmanager-values.yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: 'slack-notifications'
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#k8s-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        send_resolved: true
```

```bash
# 설정 업데이트
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f alertmanager-values.yaml
```

---

## 🦥 실무 운영 팁

### 1. 리소스 사이징

Prometheus는 수집하는 메트릭 양에 따라 리소스가 크게 달라져요.

| 클러스터 규모 | CPU | Memory | Storage |
|-------------|-----|--------|---------|
| 소규모 (노드 ~10개) | 500m | 2Gi | 50Gi |
| 중규모 (노드 ~50개) | 2 core | 8Gi | 200Gi |
| 대규모 (노드 100개+) | 4 core+ | 16Gi+ | 500Gi+ |

### 2. 데이터 보존 전략

```yaml
prometheus:
  prometheusSpec:
    retention: 15d              # 기본 보존 기간
    retentionSize: "45GB"       # 크기 기반 보존
    walCompression: true        # WAL 압축으로 디스크 절약
```

장기 보존이 필요하면 **Thanos**나 **Cortex** 같은 Long-term storage 솔루션을 사용해요.

### 3. 고가용성 (HA) 구성

프로덕션에서는 Prometheus를 이중화하는 게 좋아요.

```yaml
prometheus:
  prometheusSpec:
    replicas: 2                 # Prometheus 2개 인스턴스
    replicaExternalLabelName: "prometheus_replica"
alertmanager:
  alertmanagerSpec:
    replicas: 3                 # Alertmanager 3개 (클러스터링)
```

### 4. 네임스페이스별 리소스 모니터링 쿼리 모음

```promql
# 네임스페이스별 CPU Request 대비 사용률
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace)
/
sum(kube_pod_container_resource_requests{resource="cpu"}) by (namespace)

# 네임스페이스별 메모리 사용량
sum(container_memory_working_set_bytes{container!=""}) by (namespace)

# Deployment 스케일링 판단 (HPA 연동 시 참고)
avg(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (deployment)
```

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **아키텍처** | Pull 기반 수집 → TSDB 저장 → PromQL 쿼리 → Grafana 시각화 |
| **설치** | `kube-prometheus-stack` Helm 차트로 올인원 설치 |
| **PromQL** | `rate()`, `sum()`, `topk()` 함수로 원하는 메트릭 조회 |
| **ServiceMonitor** | CRD로 스크랩 대상 선언적 관리, labels 매칭 주의! |
| **알림** | PrometheusRule → Alertmanager → Slack/Email 연동 |
| **운영** | 데이터 보존 기간, HA 구성, 리소스 사이징 고려 |

다음 포스트에서는 **로깅 실무 — EFK Stack과 Loki**를 다뤄볼게요. 모니터링이 "지금 상태"를 보여준다면, 로깅은 "무슨 일이 있었는지"를 보여줘요! 🦥
