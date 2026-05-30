# 메타 모니터링 — Mimir로 Mimir 관찰하기

Mimir 자체의 상태를 Mimir로 모니터링하는 방법입니다.
"Mimir가 Mimir를 관찰한다"는 개념을 **메타 모니터링**이라고 합니다.

---

## 메타 모니터링이란?

```
Mimir 클러스터
    │ 자체 메트릭 노출 (/metrics)
    ▼
Prometheus (또는 Prometheus Agent)
    │ remote_write
    │ X-Scope-OrgID: mimir-meta
    ▼
동일한 Mimir 클러스터 (또는 별도 Mimir)
    │
    ▼
Grafana (Mimir 공식 대시보드로 시각화)
```

> **권장**: 운영 환경에서는 메타 모니터링용 **별도 Mimir 클러스터**를 사용하세요.
> 주요 클러스터가 장애 시 메타 모니터링도 같이 다운되는 것을 방지합니다.
> 소규모 환경에서는 동일 클러스터에 별도 테넌트(`mimir-meta`)를 사용해도 됩니다.

---

## 1. Mimir 컴포넌트 메트릭 수집 설정

Mimir 각 컴포넌트는 기본적으로 `/metrics` 엔드포인트를 노출합니다.

### Prometheus ServiceMonitor (kube-prometheus-stack)

```yaml
# mimir-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mimir
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/instance: mimir
  namespaceSelector:
    matchNames:
      - monitoring
  endpoints:
    - port: http-metrics
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
```

```bash
kubectl apply -f mimir-servicemonitor.yaml
```

### Prometheus 직접 설정 (scrape_configs)

```yaml
# prometheus.yaml
scrape_configs:
  - job_name: mimir-distributor
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [monitoring]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
        regex: distributor
        action: keep
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_instance]
        regex: mimir
        action: keep

  # 동일한 패턴으로 ingester, querier, query-frontend,
  # store-gateway, compactor, ruler, alertmanager 추가
  - job_name: mimir-ingester
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [monitoring]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
        regex: ingester
        action: keep
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_instance]
        regex: mimir
        action: keep
```

---

## 2. 메타 모니터링용 Remote Write 설정

```yaml
# Mimir 메트릭을 mimir-meta 테넌트로 전송
remote_write:
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: mimir-meta
    write_relabel_configs:
      # Mimir 관련 메트릭만 전송 (cortex_, thanos_, mimir_ 접두사)
      - source_labels: [__name__]
        regex: "(cortex|thanos|mimir|grpc|go|process)_.*"
        action: keep
    queue_config:
      max_samples_per_send: 10000
      max_shards: 10
```

---

## 3. Grafana 데이터소스 추가 (메타 모니터링 전용)

```yaml
# grafana-datasources.yaml에 추가
- name: Mimir (Meta)
  type: prometheus
  url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
  access: proxy
  httpMethod: POST
  jsonData:
    prometheusType: Mimir
    httpHeaderName1: X-Scope-OrgID
  secureJsonData:
    httpHeaderValue1: mimir-meta
```

---

## 4. 공식 Mimir 모니터링 대시보드 Import

```bash
GRAFANA_URL="http://localhost:3000"

# 전체 Mimir 모니터링 대시보드 import
DASHBOARDS=(
  "15869:Mimir-Overview"
  "15870:Mimir-Reads"
  "15871:Mimir-Ruler"
  "15872:Mimir-Writes"
  "15873:Mimir-Compactor"
  "15874:Mimir-Alertmanager"
  "15875:Mimir-ObjectStore"
  "15876:Mimir-Queries"
)

for ENTRY in "${DASHBOARDS[@]}"; do
  ID="${ENTRY%%:*}"
  NAME="${ENTRY##*:}"
  echo "Importing ${NAME} (${ID})..."
  curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
    -u admin:admin123 \
    -H "Content-Type: application/json" \
    -d "{
      \"gnetId\": ${ID},
      \"overwrite\": true,
      \"inputs\": [{
        \"name\": \"DS_PROMETHEUS\",
        \"type\": \"datasource\",
        \"pluginId\": \"prometheus\",
        \"value\": \"Mimir (Meta)\"
      }]
    }" | jq '.imported'
done
```

---

## 5. 핵심 모니터링 메트릭

### 쓰기 경로 (Write Path)

```promql
# 초당 수신 샘플 수
sum(rate(cortex_distributor_received_samples_total[5m]))

# 초당 거부된 샘플 수 (0이 정상)
sum(rate(cortex_discarded_samples_total[5m])) by (reason)

# Ingester 활성 시계열 수 (테넌트별)
sum(cortex_ingester_memory_series) by (user)

# WAL replay 지연 (파드 재시작 시 증가)
cortex_ingester_wal_replay_duration_seconds
```

### 읽기 경로 (Read Path)

```promql
# 초당 쿼리 수
sum(rate(cortex_request_duration_seconds_count{route=~".*query.*"}[5m]))

# 쿼리 P99 레이턴시 (1초 이하가 권장)
histogram_quantile(0.99,
  sum by (le) (rate(cortex_request_duration_seconds_bucket{route="api_v1_query_range"}[5m]))
)

# 쿼리 에러율
sum(rate(cortex_request_duration_seconds_count{status_code=~"5.."}[5m]))
  /
sum(rate(cortex_request_duration_seconds_count[5m]))

# 캐시 히트율 (높을수록 좋음)
sum(rate(cortex_cache_hits_total[5m]))
  /
sum(rate(cortex_cache_fetched_total[5m]))
```

### 스토리지 (S3)

```promql
# S3 요청 지연 P99
histogram_quantile(0.99,
  sum by (operation, le) (
    rate(thanos_objstore_bucket_operation_duration_seconds_bucket[5m])
  )
)

# S3 오류율
sum(rate(thanos_objstore_bucket_operation_failures_total[5m])) by (operation)

# 테넌트별 블록 수
cortex_bucket_blocks_count

# 삭제 대상 블록 수 (감소 추세여야 정상)
cortex_bucket_blocks_marked_for_deletion_total
```

### 컴팩션

```promql
# 마지막 성공 컴팩션 이후 경과 시간 (3600초 이상이면 경보)
time() - cortex_compactor_last_successful_run_timestamp_seconds

# 컴팩션 실패 수
increase(cortex_compactor_runs_failed_total[1h])
```

---

## 6. 핵심 알림 규칙 (메타 모니터링용)

```yaml
# mimir-meta-alerts.yaml
groups:
  - name: mimir.health
    rules:
      # Ingester가 S3에 flush 못하는 경우
      - alert: MimirIngesterS3Failure
        expr: |
          rate(thanos_objstore_bucket_operation_failures_total{
            component="ingester"
          }[5m]) > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Mimir Ingester S3 업로드 실패"

      # Ingestion rate limit으로 샘플 유실
      - alert: MimirSamplesDiscarded
        expr: |
          sum(rate(cortex_discarded_samples_total[5m])) by (user) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "테넌트 {{ $labels.user }} 샘플 유실 중"
          description: "초당 {{ $value | humanize }} 샘플이 거부되고 있습니다."

      # Compactor 중단
      - alert: MimirCompactorNotRunning
        expr: |
          time() - cortex_compactor_last_successful_run_timestamp_seconds > 3600
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Mimir Compactor 1시간 이상 미실행"

      # Ingester 링 degraded
      - alert: MimirIngesterRingDegraded
        expr: |
          sum(cortex_ring_members{name="ingester", state="ACTIVE"}) < 3
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Mimir Ingester 링 degraded (ACTIVE < 3)"
          description: "현재 ACTIVE Ingester 수: {{ $value }}"

      # 쿼리 에러율 높음
      - alert: MimirHighQueryErrorRate
        expr: |
          sum(rate(cortex_request_duration_seconds_count{status_code=~"5.."}[5m]))
            /
          sum(rate(cortex_request_duration_seconds_count[5m]))
          > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Mimir 쿼리 에러율 1% 초과"
```

---

## 7. 메타 모니터링 알림 업로드

```bash
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring

# mimir-meta 테넌트로 알림 규칙 업로드
curl -X POST \
  http://localhost:8080/prometheus/config/v1/rules/mimir-health \
  -H "X-Scope-OrgID: mimir-meta" \
  -H "Content-Type: application/yaml" \
  --data-binary @mimir-meta-alerts.yaml
```

---

## 참고 링크

- [Mimir 메타 모니터링 공식 문서](https://grafana.com/docs/mimir/latest/manage/monitor-grafana-mimir/)
- [Mimir Grafana 대시보드 목록](https://grafana.com/grafana/dashboards/?search=mimir)
