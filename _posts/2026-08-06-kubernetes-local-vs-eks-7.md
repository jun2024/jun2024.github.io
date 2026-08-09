---
title: "[Kubernetes] 로컬 K8s vs EKS (7) — 모니터링 & 로깅: Prometheus vs CloudWatch"
excerpt: "K8s 모니터링/로깅 도구가 환경에 따라 어떻게 달라질까? Prometheus+Grafana vs CloudWatch Container Insights, EFK vs CloudWatch Logs 비교 정리!"

categories:
  - Kubernetes
tags:
  - [Kubernetes, K8s, EKS, AWS, Monitoring, Logging, Prometheus, CloudWatch, DevOps]

permalink: /kubernetes/local-vs-eks-7/

toc: true
toc_sticky: true

date: 2026-08-06
last_modified_at: 2026-08-06
---

> 이전 포스트 [로컬 vs EKS (6)](/kubernetes/local-vs-eks-6/)에서 업그레이드 & 유지보수 차이를 다뤘어요!

---

## 🦥 모니터링 & 로깅 — 도구 선택이 달라진다

K8s 모니터링/로깅의 **개념**은 동일하지만, **어떤 도구를 사용하느냐**가 환경에 따라 달라요.

| 영역 | 로컬 K8s | EKS 옵션 |
|------|---------|----------|
| 메트릭 수집 | Metrics Server + Prometheus | CloudWatch Container Insights / Prometheus |
| 메트릭 시각화 | Grafana | CloudWatch Dashboard / Grafana |
| 로그 수집 | EFK (Elasticsearch+Fluentd+Kibana) | CloudWatch Logs / EFK |
| 알림 | Alertmanager | CloudWatch Alarms / Alertmanager |

---

## 🦥 메트릭 모니터링

### 로컬 K8s — Prometheus + Grafana 직접 구축

[모니터링 포스트](/kubernetes/kubernetes-monitoring-prometheus-grafana/)에서 다뤘던 구성이에요.

```bash
# Prometheus + Grafana 설치 (Helm)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

| 구성 요소 | 역할 |
|----------|------|
| Prometheus | 메트릭 수집 (Pull 방식) |
| Grafana | 대시보드, 시각화 |
| Alertmanager | 알림 (Slack, PagerDuty 등) |
| node-exporter | 노드 메트릭 |
| kube-state-metrics | K8s 리소스 상태 메트릭 |

| 장점 | 부담 |
|------|------|
| 완전한 커스터마이징 | 직접 운영/유지보수 |
| PromQL로 강력한 쿼리 | 스토리지 관리 (장기 보관) |
| 풍부한 커뮤니티 대시보드 | 고가용성 직접 구성 |

### EKS 옵션 1 — CloudWatch Container Insights

AWS 네이티브 모니터링 도구예요. 별도 인프라 없이 바로 사용할 수 있어요.

```bash
# CloudWatch Agent + Fluent Bit 설치 (한 번에)
ClusterName=my-cluster
RegionName=ap-northeast-2
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'

curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | \
  sed "s/{{cluster_name}}/${ClusterName}/;s/{{region_name}}/${RegionName}/" | \
  kubectl apply -f -
```

| CloudWatch 메트릭 | 설명 |
|------------------|------|
| 클러스터 레벨 | CPU, 메모리, 네트워크, 노드 수 |
| 노드 레벨 | 노드별 리소스 사용률 |
| Pod 레벨 | Pod별 CPU, 메모리, 재시작 횟수 |
| 컨테이너 레벨 | 컨테이너별 상세 메트릭 |

| 장점 | 한계 |
|------|------|
| 설치 간편 (DaemonSet) | PromQL 미지원 (Metrics Insights 쿼리) |
| AWS 서비스 통합 (RDS, ALB 등) | 대시보드 커스터마이징 제한 |
| 관리 부담 없음 | 비용 (로그 수집량, 메트릭 수) |
| CloudWatch Alarms 연동 | 커뮤니티 대시보드 없음 |

### EKS 옵션 2 — Amazon Managed Prometheus + Grafana

Prometheus를 직접 운영하지 않고 **AWS가 관리하는 Prometheus**를 사용할 수 있어요.

```bash
# Amazon Managed Prometheus 워크스페이스 생성
aws amp create-workspace --alias my-prometheus

# Prometheus 서버 설정 (remote_write로 AMP에 전송)
# Helm으로 Prometheus 설치 시 remote_write 설정 추가
helm install prometheus prometheus-community/prometheus \
  --set server.remoteWrite[0].url="https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/<workspace-id>/api/v1/remote_write" \
  --set server.remoteWrite[0].sigv4.region="ap-northeast-2"
```

```bash
# Amazon Managed Grafana 워크스페이스 생성
aws grafana create-workspace \
  --workspace-name my-grafana \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO
```

| 비교 | 직접 구축 Prometheus | Amazon Managed Prometheus |
|------|-------------------|-------------------------|
| 운영 | 직접 (스토리지, HA) | AWS 관리 |
| 확장성 | 직접 구성 | 자동 확장 |
| 장기 보관 | Thanos/Cortex 필요 | 150일 기본 보관 |
| 비용 | EC2 리소스 | 수집 샘플 + 쿼리 + 보관량 |
| PromQL | 지원 | 지원 |
| Grafana 연동 | 직접 구축 | Managed Grafana 연동 |

---

## 🦥 로깅

### 로컬 K8s — EFK Stack

[로깅 포스트](/kubernetes/kubernetes-logging-efk-loki/)에서 다뤘던 구성이에요.

| 구성 요소 | 역할 |
|----------|------|
| **Elasticsearch** | 로그 저장/검색 |
| **Fluentd/Fluent Bit** | 로그 수집 (DaemonSet) |
| **Kibana** | 로그 시각화 |

| 장점 | 부담 |
|------|------|
| 강력한 검색/분석 | Elasticsearch 클러스터 운영 |
| Kibana 시각화 | 높은 리소스 사용량 |
| 완전한 커스터마이징 | 인덱스 관리, 보관 정책 |

### EKS 옵션 1 — CloudWatch Logs

Container Insights를 설치하면 **로그도 함께** CloudWatch로 전송돼요.

```bash
# Pod 로그 확인 (CloudWatch Logs Insights)
# CloudWatch Console → Logs Insights에서 쿼리
```

```
# CloudWatch Logs Insights 쿼리 예시
fields @timestamp, @message, kubernetes.pod_name
| filter kubernetes.namespace_name = "my-app"
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50
```

| 장점 | 한계 |
|------|------|
| 별도 인프라 불필요 | 비용 (수집 $0.50/GB, 보관 $0.03/GB/월) |
| AWS 서비스 로그 통합 | Kibana보다 분석 기능 제한 |
| Log Insights로 쿼리 | 대량 로그 시 비용 급증 |
| 보관 기간 설정 간편 | 실시간 스트리밍 제한 |

### EKS 옵션 2 — AWS OpenSearch (구 Elasticsearch)

EFK의 Elasticsearch를 AWS 관리형으로 대체할 수 있어요.

```bash
# Fluent Bit → OpenSearch로 전송 설정
# Fluent Bit DaemonSet ConfigMap에서 output 설정
[OUTPUT]
    Name            es
    Match           *
    Host            search-my-domain.ap-northeast-2.es.amazonaws.com
    Port            443
    TLS             On
    AWS_Auth        On
    AWS_Region      ap-northeast-2
    Index           k8s-logs
```

| 비교 | 자체 구축 EFK | AWS OpenSearch |
|------|-------------|---------------|
| 운영 | 직접 (클러스터 관리) | AWS 관리 |
| Kibana/Dashboard | 직접 설치 | OpenSearch Dashboards 포함 |
| 확장 | 수동 | 간편 (인스턴스 추가) |
| 비용 | EC2 리소스 | 인스턴스 + 스토리지 과금 |

---

## 🦥 모니터링/로깅 조합 추천

### 소규모 팀 / 시작 단계

| 영역 | 추천 | 이유 |
|------|------|------|
| 메트릭 | CloudWatch Container Insights | 설치 간편, 관리 부담 최소 |
| 로그 | CloudWatch Logs | 통합 관리, 추가 인프라 불필요 |
| 알림 | CloudWatch Alarms + SNS | AWS 네이티브 |

### 중규모 팀 / 성장 단계

| 영역 | 추천 | 이유 |
|------|------|------|
| 메트릭 | Amazon Managed Prometheus + Grafana | PromQL, 커스텀 대시보드 |
| 로그 | CloudWatch Logs 또는 Loki | 비용과 기능 밸런스 |
| 알림 | Alertmanager + PagerDuty | 세밀한 알림 규칙 |

### 대규모 / 멀티 클러스터

| 영역 | 추천 | 이유 |
|------|------|------|
| 메트릭 | Prometheus + Thanos / Managed Prometheus | 장기 보관, 멀티 클러스터 |
| 로그 | OpenSearch 또는 Loki | 대량 로그 검색/분석 |
| 알림 | Alertmanager + OpsGenie | 온콜 연동 |

---

## 🦥 비용 비교 (월 기준 예시)

소규모 클러스터(노드 3개, Pod 50개) 기준 대략적인 비용이에요.

| 구성 | 월 비용 (대략) | 관리 부담 |
|------|--------------|----------|
| CloudWatch (Insights + Logs) | $50~150 | 낮음 |
| 자체 구축 (Prometheus + EFK) | $0 (EC2 리소스 내) | 높음 |
| Managed Prometheus + Grafana | $100~300 | 중간 |
| Datadog / New Relic (SaaS) | $300~1000+ | 최소 |

> 💡 **실무 팁**: 처음에는 CloudWatch로 시작하고, 모니터링 요구사항이 복잡해지면 Prometheus로 전환하는 팀이 많아요. 둘을 병행하는 것도 가능해요.

---

## 🦥 EKS에서도 kubectl top은 동일

```bash
# Metrics Server는 EKS에서도 동일하게 동작
kubectl top nodes
kubectl top pods -A

# 특정 Pod 로그 확인도 동일
kubectl logs <pod-name> -f
kubectl logs <pod-name> -c <container-name> --previous
```

CKA에서 배운 `kubectl top`, `kubectl logs` 명령어는 EKS에서도 그대로 사용해요. 차이는 **장기 보관과 분석 도구**에 있어요.

---

## 🦥 시리즈 전체 정리

7편에 걸쳐 로컬 K8s와 EKS의 차이를 비교했어요. 핵심을 한 테이블로 요약하면 이래요.

| 영역 | 로컬 K8s (직접 관리) | EKS (AWS 관리형) |
|------|-------------------|-----------------|
| Control Plane | 직접 관리 | AWS 관리 |
| 설치 | kubeadm (수동) | eksctl / Terraform |
| CNI | Flannel, Calico (Overlay) | VPC CNI (네이티브) |
| Service 노출 | MetalLB | AWS LB 자동 생성 |
| Ingress | Nginx Ingress | ALB Controller |
| 스토리지 | hostPath, NFS | EBS, EFS (CSI) |
| 인증 | 인증서 기반 | IAM + aws-auth |
| Pod→AWS | 해당 없음 | IRSA / Pod Identity |
| 업그레이드 | kubeadm upgrade | AWS 관리 + Node Group |
| 백업 | etcd snapshot | Velero + S3 |
| 모니터링 | Prometheus + Grafana | CloudWatch / Managed Prometheus |
| 로깅 | EFK | CloudWatch Logs / OpenSearch |

> 💡 **결론**: CKA에서 배운 K8s 개념은 EKS에서도 동일하게 적용돼요. 달라지는 건 **인프라 계층의 구현 방식**이에요. K8s 기초를 탄탄히 하면 어떤 환경에서든 빠르게 적응할 수 있어요. 🦥
