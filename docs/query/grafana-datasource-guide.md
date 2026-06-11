# Read Path — Grafana 연동

Grafana에서 Mimir를 데이터소스로 추가하여 PromQL로 메트릭을 조회합니다.

---

## 데이터소스 유형 선택

| 유형 | 권장 상황 |
|---|---|
| **Prometheus** 타입 | Grafana < 9.x 또는 기존 대시보드 호환 필요 |
| **Mimir** 타입 (native) | Grafana >= 9.x, Ruler/Alertmanager UI 통합 사용 시 |

---

## Grafana UI에서 추가

### Prometheus 타입 (범용)

1. Grafana → **Configuration** → **Data Sources** → **Add data source**
2. **Prometheus** 선택
3. 아래 설정 입력:

| 항목 | 값 |
|---|---|
| Name | `Mimir` |
| URL | `http://mimir-nginx.monitoring.svc.cluster.local/prometheus` |
| HTTP Method | `POST` |

4. **Custom HTTP Headers** 추가:
   - Header: `X-Scope-OrgID`
   - Value: `default` (또는 원하는 테넌트 ID)

5. **Prometheus type** → `Mimir` 선택 (Grafana 9+)

6. **Save & Test** 클릭

---

## Grafana as Code (Provisioning)

### 단일 테넌트

```yaml
# grafana/provisioning/datasources/mimir.yaml
apiVersion: 1

datasources:
  - name: Mimir
    type: prometheus
    url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
    access: proxy
    httpMethod: POST
    jsonData:
      prometheusType: Mimir
      prometheusVersion: "2.9.0"
      httpHeaderName1: X-Scope-OrgID
      timeInterval: "15s"          # scrape_interval과 일치
      queryTimeout: "60s"
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: tempo
    secureJsonData:
      httpHeaderValue1: default
    isDefault: true
    editable: true
```

### 멀티 테넌트 (팀별 데이터소스)

```yaml
apiVersion: 1

datasources:
  - name: Mimir (team-a)
    type: prometheus
    uid: mimir-team-a
    url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
    access: proxy
    httpMethod: POST
    jsonData:
      prometheusType: Mimir
      httpHeaderName1: X-Scope-OrgID
    secureJsonData:
      httpHeaderValue1: team-a

  - name: Mimir (team-b)
    type: prometheus
    uid: mimir-team-b
    url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
    access: proxy
    httpMethod: POST
    jsonData:
      prometheusType: Mimir
      httpHeaderName1: X-Scope-OrgID
    secureJsonData:
      httpHeaderValue1: team-b
```

---

## Grafana Helm Chart로 배포 시

```yaml
# grafana-values.yaml
grafana:
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Mimir
          type: prometheus
          url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
          access: proxy
          httpMethod: POST
          jsonData:
            prometheusType: Mimir
            httpHeaderName1: X-Scope-OrgID
          secureJsonData:
            httpHeaderValue1: default
          isDefault: true
```

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values grafana-values.yaml
```

---

## 쿼리 확인

Grafana Explore 탭에서 아래 쿼리로 연동 확인:

```promql
# 수집 중인 메트릭 종류 수
count(count by (__name__) ({__name__=~".+"}))

# Mimir ingester 수신 샘플 수 확인 (메타 모니터링)
rate(cortex_ingester_ingested_samples_total[5m])

# up 메트릭 (scrape 성공 여부)
up

# 현재 활성 시계열 수
cortex_ingester_memory_series
```

---

## 공식 Mimir 모니터링 대시보드 Import

Grafana에서 아래 대시보드 ID로 import하세요:

| 대시보드 이름 | Grafana ID | 용도 |
|---|---|---|
| Mimir / Overview | 15869 | 전체 상태 한눈에 보기 |
| Mimir / Writes | 15872 | Distributor/Ingester 쓰기 지표 |
| Mimir / Reads | 15870 | Querier/Query Frontend 읽기 지표 |
| Mimir / Compactor | 15873 | Compaction 상태 |
| Mimir / Ruler | 15874 | Recording/Alerting Rules 상태 |
| Mimir / Alertmanager | 15875 | Alertmanager 상태 |
| Mimir / Object Store | 15876 | S3 접근 지표 |

```bash
# Grafana API로 대시보드 일괄 import
GRAFANA_URL="http://localhost:3000"
GRAFANA_TOKEN="your-api-key"

for DASHBOARD_ID in 15869 15870 15872 15873 15874 15875 15876; do
  curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
    -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{
      \"gnetId\": ${DASHBOARD_ID},
      \"overwrite\": true,
      \"inputs\": [{
        \"name\": \"DS_PROMETHEUS\",
        \"type\": \"datasource\",
        \"pluginId\": \"prometheus\",
        \"value\": \"Mimir\"
      }]
    }"
done
```

> **주의**: 위 대시보드는 Mimir 자체를 모니터링하는 메타 모니터링용입니다.
> Mimir가 자기 자신의 메트릭을 수집하도록 설정해야 합니다.
> 자세한 내용은 [monitoring-guide.md](../operations/monitoring-guide.md) 참고.

---

## 쿼리 성능 최적화

### Query Frontend 캐싱 활성화

```yaml
mimir:
  structuredConfig:
    query_range:
      cache_results: true
      results_cache:
        backend: memcached
        memcached:
          addresses: dns+mimir-memcached.monitoring.svc:11211
          max_idle_connections: 16
          max_async_concurrency: 50
          max_get_multi_concurrency: 100
```

### 쿼리 샤딩 (Query Sharding)

Query Frontend가 고카디널리티 집계 쿼리를 자동으로 병렬 실행합니다:

```yaml
mimir:
  structuredConfig:
    frontend:
      parallelize_shardable_queries: true
    query_sharding_total_shards: 16
```

### 쿼리 타임아웃

```yaml
mimir:
  structuredConfig:
    limits:
      query_timeout: 120s          # 기본 60s
      max_query_parallelism: 32
```

---

## Recording Rules 활용 (쿼리 최적화)

Grafana 대시보드가 느릴 때 Recording Rules로 미리 계산:

```yaml
# Grafana 대시보드의 느린 쿼리를 Recording Rule로 대체
groups:
  - name: dashboard.rules
    interval: 1m
    rules:
      # 대시보드에서 자주 사용하는 복잡한 쿼리
      - record: job:http_request_duration_p99:1m
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| `No data` | 테넌트 헤더 불일치 | `X-Scope-OrgID` 값 확인 |
| `context deadline exceeded` | 쿼리 타임아웃 | 쿼리 범위 축소, Recording Rule 사용 |
| `vector selector must contain at least one non-empty matcher` | PromQL 문법 오류 | 레이블 matcher 확인 |
| 느린 쿼리 | 캐시 미사용 | Query Frontend 캐싱 활성화 |
| `data source not responding` | URL 또는 서비스 이름 오타 | mimir-nginx 서비스 이름/포트 확인 |

```bash
# Query Frontend 로그에서 느린 쿼리 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=query-frontend \
  | grep "slow query\|duration" | tail -20

# 쿼리 히트율 확인 (캐시 효율)
curl "http://localhost:8080/prometheus/api/v1/query?query=rate(cortex_cache_hits_total[5m])/rate(cortex_cache_fetched_total[5m])" \
  -H "X-Scope-OrgID: default" | jq
```
