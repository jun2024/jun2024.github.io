---
title: "[Kubernetes] K8s 로깅 실무 — EFK Stack과 Loki로 중앙 로그 관리하기"
excerpt: "Elasticsearch + Fluentd + Kibana(EFK)와 Grafana Loki — 두 가지 로깅 스택의 아키텍처, 설치, 실무 운영 방법을 비교하며 정리해보자!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, Logging, EFK, Loki, Fluentd, Elasticsearch, DevOps]

permalink: /kubernetes/kubernetes-logging-efk-loki/

toc: true
toc_sticky: true

date: 2026-04-14
last_modified_at: 2026-04-14
---

> 이전 포스트에서 [모니터링 기본 개념](/kubernetes/kubernetes-monitoring-logging-basics/)과 [Prometheus + Grafana 실무](/kubernetes/kubernetes-monitoring-prometheus-grafana/)를 다뤘어요.
> 이번에는 **로깅 실무** — 로그를 중앙에서 수집하고 검색하는 방법을 알아볼게요!

---

## 🦥 왜 중앙 로그 관리가 필요할까?

`kubectl logs`로 개별 Pod 로그를 볼 수 있지만, 실무에서는 이것만으로 부족해요.

| 상황 | kubectl logs | 중앙 로그 시스템 |
|------|-------------|----------------|
| Pod이 삭제/재시작됨 | 로그 유실 | 보존됨 |
| 여러 Pod의 로그를 동시에 검색 | 하나씩 확인 | 통합 검색 |
| 과거 특정 시간대 로그 | 불가능 | 시간 범위 필터링 |
| 로그 패턴 분석/시각화 | 불가능 | 대시보드 제공 |
| 알림 (에러 급증 등) | 불가능 | 알림 설정 가능 |

대표적인 K8s 로깅 스택은 **EFK (Elasticsearch + Fluentd + Kibana)**와 **Loki + Grafana** 두 가지예요.

---

## 🦥 EFK vs Loki — 어떤 걸 선택할까?

| 비교 항목 | EFK Stack | Loki + Grafana |
|----------|-----------|----------------|
| **로그 저장** | 풀텍스트 인덱싱 (Elasticsearch) | 라벨 기반 인덱싱, 로그 본문은 압축 저장 |
| **검색 성능** | 매우 강력 (전문 검색) | 라벨 기반 필터 후 grep 방식 |
| **리소스 사용량** | 높음 (ES가 메모리 많이 소비) | 낮음 (경량) |
| **운영 복잡도** | 높음 (ES 클러스터 관리) | 낮음 |
| **비용** | 높음 (스토리지 + 컴퓨팅) | 낮음 |
| **Grafana 통합** | 별도 Kibana 필요 | Grafana에서 바로 조회 |
| **적합한 환경** | 대규모, 복잡한 검색 필요 | 중소규모, 비용 효율 중시 |

> 💡 최근 트렌드는 **Loki**로 이동하는 추세예요. Prometheus + Grafana 모니터링 스택과 자연스럽게 통합되고, 운영 비용이 훨씬 적어요. 하지만 복잡한 전문 검색이 필요하면 EFK가 여전히 강력해요.

---

## 🦥 EFK Stack 아키텍처

```
┌──────────────────────────────────────────────────┐
│                 Kubernetes Cluster                │
│                                                  │
│  ┌──────┐ ┌──────┐ ┌──────┐                     │
│  │Pod A │ │Pod B │ │Pod C │  stdout/stderr       │
│  └──┬───┘ └──┬───┘ └──┬───┘                     │
│     └────────┼────────┘                          │
│              ▼                                   │
│     /var/log/containers/*.log                    │
│              │                                   │
│  ┌───────────▼────────────┐                      │
│  │  Fluentd (DaemonSet)   │  각 노드마다 1개     │
│  │  - 로그 수집            │                      │
│  │  - 파싱/필터링          │                      │
│  │  - 태그 추가            │                      │
│  └───────────┬────────────┘                      │
│              │                                   │
│  ┌───────────▼────────────┐                      │
│  │    Elasticsearch       │                      │
│  │  - 풀텍스트 인덱싱     │                      │
│  │  - 시계열 저장          │                      │
│  │  - 검색 엔진            │                      │
│  └───────────┬────────────┘                      │
│              │                                   │
│  ┌───────────▼────────────┐                      │
│  │       Kibana           │                      │
│  │  - 로그 검색/시각화     │                      │
│  │  - 대시보드             │                      │
│  └────────────────────────┘                      │
└──────────────────────────────────────────────────┘
```

### Elasticsearch 설치

```bash
# ECK (Elastic Cloud on Kubernetes) Operator 설치
kubectl create -f https://download.elastic.co/downloads/eck/2.11.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.11.1/operator.yaml
```

```yaml
# elasticsearch.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: logging
spec:
  version: 8.12.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 4Gi
              cpu: "1"
            limits:
              memory: 4Gi
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: gp3
```

### Fluentd DaemonSet 설치

Fluentd는 각 노드에 DaemonSet으로 배포해서 컨테이너 로그를 수집해요.

```yaml
# fluentd-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule      # Control Plane 노드에도 배포
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch-es-http.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "https"
        - name: FLUENT_ELASTICSEARCH_SSL_VERIFY
          value: "false"
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            memory: 512Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: containers
          mountPath: /var/log/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containers
        hostPath:
          path: /var/log/containers
```

### Fluentd 설정 (ConfigMap)

```yaml
# fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    # 컨테이너 로그 수집
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_key time
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    # K8s 메타데이터 추가 (pod, namespace, labels 등)
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
    </filter>

    # 특정 네임스페이스 제외 (kube-system 로그 제외 예시)
    <filter kubernetes.**>
      @type grep
      <exclude>
        key $.kubernetes.namespace_name
        pattern /^(kube-system|kube-public)$/
      </exclude>
    </filter>

    # Elasticsearch로 전송
    <match kubernetes.**>
      @type elasticsearch
      host elasticsearch-es-http.logging.svc.cluster.local
      port 9200
      scheme https
      ssl_verify false
      logstash_format true
      logstash_prefix k8s-logs
      include_tag_key true
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        flush_interval 5s
        retry_max_interval 30
        chunk_limit_size 2M
        total_limit_size 500M
        overflow_action block
      </buffer>
    </match>
```

### Kibana 설치

```yaml
# kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: logging
spec:
  version: 8.12.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
```

```bash
# Kibana 접근
kubectl port-forward svc/kibana-kb-http 5601:5601 -n logging

# 비밀번호 확인
kubectl get secret elasticsearch-es-elastic-user -n logging -o jsonpath='{.data.elastic}' | base64 -d
```

---

## 🦥 Loki Stack 아키텍처

Loki는 Grafana Labs에서 만든 경량 로그 수집 시스템이에요. "Prometheus for logs"라고 불리며, 라벨 기반으로 로그를 인덱싱해요.

```
┌──────────────────────────────────────────────────┐
│                 Kubernetes Cluster                │
│                                                  │
│  ┌──────┐ ┌──────┐ ┌──────┐                     │
│  │Pod A │ │Pod B │ │Pod C │  stdout/stderr       │
│  └──┬───┘ └──┬───┘ └──┬───┘                     │
│     └────────┼────────┘                          │
│              ▼                                   │
│     /var/log/containers/*.log                    │
│              │                                   │
│  ┌───────────▼────────────┐                      │
│  │  Promtail (DaemonSet)  │  각 노드마다 1개     │
│  │  - 로그 수집            │                      │
│  │  - 라벨 추가            │                      │
│  └───────────┬────────────┘                      │
│              │                                   │
│  ┌───────────▼────────────┐                      │
│  │       Loki             │                      │
│  │  - 라벨 인덱싱         │                      │
│  │  - 로그 청크 압축 저장  │                      │
│  └───────────┬────────────┘                      │
│              │                                   │
│  ┌───────────▼────────────┐                      │
│  │       Grafana          │  ← Prometheus와      │
│  │  - LogQL 쿼리           │    같은 UI에서 조회  │
│  │  - 로그 + 메트릭 통합    │                      │
│  └────────────────────────┘                      │
└──────────────────────────────────────────────────┘
```

### Loki 설치 (Helm)

```bash
# Grafana Helm 레포 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Loki Stack 설치 (Loki + Promtail)
kubectl create namespace logging

helm install loki grafana/loki-stack \
  --namespace logging \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi \
  --set loki.persistence.storageClassName=gp3
```

### Loki 설치 확인

```bash
kubectl get pods -n logging
```

```
NAME                           READY   STATUS    RESTARTS
loki-0                         1/1     Running   0
loki-promtail-abcde            1/1     Running   0
loki-promtail-fghij            1/1     Running   0
loki-promtail-klmno            1/1     Running   0
```

### Grafana에 Loki 데이터소스 추가

이미 Prometheus + Grafana를 설치했다면, Loki를 데이터소스로 추가만 하면 돼요.

```yaml
# grafana-datasource.yaml (Helm values)
grafana:
  additionalDataSources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki.logging.svc.cluster.local:3100
    jsonData:
      maxLines: 1000
```

또는 Grafana UI에서 직접 추가할 수 있어요: **Configuration → Data Sources → Add data source → Loki** → URL에 `http://loki.logging.svc.cluster.local:3100` 입력.

---

## 🦥 LogQL — Loki 쿼리 언어

LogQL은 PromQL과 비슷한 문법으로 로그를 검색할 수 있어요. Grafana의 Explore 메뉴에서 바로 사용할 수 있어요.

### 기본 쿼리

```logql
# 특정 네임스페이스의 로그
{namespace="default"}

# 특정 Pod 로그
{namespace="default", pod="webapp-abc123"}

# 특정 컨테이너 로그
{namespace="default", container="nginx"}

# 여러 조건 조합
{namespace="production", app="api-server"}
```

### 필터링

```logql
# 로그 본문에 "error" 포함
{namespace="default"} |= "error"

# 대소문자 무시 검색
{namespace="default"} |~ "(?i)error"

# "error" 포함하되 "timeout" 제외
{namespace="default"} |= "error" != "timeout"

# 정규식 매칭
{namespace="default"} |~ "status=[45]\\d{2}"
```

### 로그 파싱 및 필터링

```logql
# JSON 로그 파싱
{app="api-server"} | json | status >= 400

# logfmt 로그 파싱
{app="api-server"} | logfmt | duration > 1s

# 특정 필드 추출 후 필터
{app="nginx"} | pattern `<ip> - - [<timestamp>] "<method> <path> <_>" <status> <size>` | status = "500"
```

### 메트릭 쿼리 (로그에서 메트릭 추출)

```logql
# 최근 5분간 에러 로그 건수 (초당)
rate({namespace="default"} |= "error" [5m])

# 네임스페이스별 에러 로그 건수
sum(rate({namespace=~".+"} |= "error" [5m])) by (namespace)

# 상태코드 500 로그 건수
sum(count_over_time({app="nginx"} | json | status = 500 [1h])) by (pod)
```

> 💡 LogQL의 메트릭 쿼리를 활용하면 별도 코드 수정 없이 **로그 기반 알림**을 설정할 수 있어요!

---

## 🦥 Loki 알림 설정

Loki의 **Ruler** 컴포넌트를 사용하면 로그 기반 알림을 설정할 수 있어요.

```yaml
# loki-ruler-config (Helm values)
loki:
  rulerConfig:
    alertmanager_url: http://kube-prometheus-stack-alertmanager.monitoring.svc.cluster.local:9093
    ring:
      kvstore:
        store: inmemory
    rule_path: /tmp/rules
    storage:
      type: local
      local:
        directory: /etc/loki/rules
  rules:
    - name: error-alerts
      rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({namespace="production"} |= "error" [5m])) by (app) > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.app }}에서 에러 로그가 급증했어요 (분당 10건 이상)"
      - alert: PodOOMKilled
        expr: |
          sum(rate({namespace=~".+"} |= "OOMKilled" [5m])) by (namespace) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.namespace }}에서 OOM Kill이 발생했어요"
```

---

## 🦥 Fluent Bit vs Fluentd

EFK에서 수집기로 Fluentd 대신 **Fluent Bit**을 사용하는 경우도 많아요.

| 비교 | Fluentd | Fluent Bit |
|------|---------|------------|
| **언어** | Ruby + C | C |
| **메모리 사용량** | ~60MB | ~5MB |
| **플러그인 수** | 700+ | 100+ |
| **적합한 역할** | 로그 집계/가공 (Aggregator) | 로그 수집 (Collector) |
| **K8s 배포** | DaemonSet | DaemonSet |

실무에서는 이런 조합을 많이 사용해요.

```
각 노드: Fluent Bit (경량 수집)
    ↓
중앙: Fluentd (집계/가공/라우팅)
    ↓
저장소: Elasticsearch 또는 Loki
```

---

## 🦥 실무 운영 팁

### 1. 로그 볼륨 관리

로그가 너무 많으면 스토리지 비용과 검색 성능에 영향을 줘요.

```yaml
# Fluentd에서 불필요한 로그 필터링
<filter kubernetes.**>
  @type grep
  <exclude>
    key $.kubernetes.namespace_name
    pattern /^(kube-system|kube-public|kube-node-lease)$/
  </exclude>
</filter>

# Health check 로그 제외
<filter kubernetes.**>
  @type grep
  <exclude>
    key log
    pattern /GET \/healthz|GET \/readyz|GET \/livez/
  </exclude>
</filter>
```

### 2. 구조화된 로그 출력 (JSON)

애플리케이션에서 JSON 형태로 로그를 출력하면 파싱과 검색이 훨씬 편해요.

```python
# 비추천: 비구조화 로그
logger.info("User login successful - user_id: 12345")

# 추천: 구조화 로그 (JSON)
import json
logger.info(json.dumps({
    "event": "user_login",
    "status": "success",
    "user_id": 12345,
    "ip": "10.0.1.50",
    "duration_ms": 45
}))
```

### 3. 멀티 테넌시

여러 팀이 같은 로깅 시스템을 사용할 때는 접근 제어가 중요해요.

| 솔루션 | 멀티 테넌시 방법 |
|--------|---------------|
| **Elasticsearch** | Index 기반 분리 + Kibana Spaces |
| **Loki** | Tenant Header (`X-Scope-OrgID`) |

### 4. 로그 보존 정책

```yaml
# Loki 보존 설정
loki:
  config:
    table_manager:
      retention_deletes_enabled: true
      retention_period: 720h    # 30일
    limits_config:
      retention_period: 720h
```

```
# Elasticsearch ILM (Index Lifecycle Management) 예시
PUT _ilm/policy/k8s-logs-policy
{
  "policy": {
    "phases": {
      "hot":    { "min_age": "0ms",  "actions": { "rollover": { "max_size": "50GB", "max_age": "1d" }}},
      "warm":   { "min_age": "7d",   "actions": { "shrink": { "number_of_shards": 1 }}},
      "cold":   { "min_age": "30d",  "actions": { "freeze": {} }},
      "delete": { "min_age": "90d",  "actions": { "delete": {} }}
    }
  }
}
```

---

## 🦥 전체 Observability 스택 정리

3개 포스트에 걸쳐 다룬 전체 K8s Observability 스택을 정리해볼게요.

```
┌─────────────────────────────────────────────────────────┐
│                  K8s Observability Stack                 │
│                                                         │
│  ┌───────────────────┐  ┌────────────────────┐         │
│  │    Monitoring       │  │     Logging         │         │
│  │                     │  │                     │         │
│  │  Metrics Server     │  │  kubectl logs       │         │
│  │  (CKA 기본)         │  │  (CKA 기본)         │         │
│  │         ↓           │  │         ↓            │         │
│  │  Prometheus         │  │  EFK Stack           │         │
│  │  + Grafana          │  │  또는 Loki + Grafana │         │
│  │  (실무 표준)        │  │  (실무 표준)         │         │
│  └───────────────────┘  └────────────────────┘         │
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │              Alerting                     │           │
│  │  Alertmanager (메트릭 기반)               │           │
│  │  + Loki Ruler (로그 기반)                 │           │
│  │  → Slack, Email, PagerDuty               │           │
│  └──────────────────────────────────────────┘           │
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │              Visualization                │           │
│  │  Grafana (모니터링 + 로깅 통합 대시보드)  │           │
│  └──────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

| 영역 | CKA 수준 | 실무 수준 |
|------|---------|----------|
| **메트릭 수집** | Metrics Server + kubectl top | Prometheus + Exporter |
| **메트릭 시각화** | - | Grafana 대시보드 |
| **로그 수집** | kubectl logs | Fluentd/Fluent Bit/Promtail |
| **로그 저장/검색** | - | Elasticsearch 또는 Loki |
| **로그 시각화** | - | Kibana 또는 Grafana |
| **알림** | - | Alertmanager + Loki Ruler |

---

## 🦥 정리

| 주제 | 핵심 내용 |
|------|----------|
| **EFK Stack** | 풀텍스트 인덱싱으로 강력한 검색, 리소스 사용량 높음 |
| **Loki + Grafana** | 라벨 기반 경량 로깅, Prometheus와 자연스러운 통합 |
| **수집기** | Fluentd (기능 풍부) vs Fluent Bit (경량), 조합 사용 가능 |
| **LogQL** | PromQL과 유사한 문법, 로그 파싱 및 메트릭 추출 가능 |
| **운영 핵심** | 로그 볼륨 관리, JSON 구조화 로그, 보존 정책 설정 |
| **트렌드** | Loki 중심으로 이동 중, Grafana에서 메트릭+로그 통합 관리 |

이것으로 K8s 모니터링 & 로깅 시리즈를 마무리할게요! CKA 기본 개념부터 Prometheus + Grafana 모니터링, EFK/Loki 로깅까지 — 실무에서 바로 적용할 수 있는 내용으로 정리했어요. 🦥
