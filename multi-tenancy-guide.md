# 멀티 테넌시 (Multi-Tenancy)

Mimir는 기본적으로 멀티 테넌트 아키텍처입니다.
모든 쓰기/읽기 요청에 **X-Scope-OrgID** 헤더를 포함해야 합니다.

---

## 테넌트란?

- 테넌트 = 독립된 메트릭 저장소 공간 (논리적 격리)
- 테넌트 간 데이터는 **완전히 격리** (다른 테넌트의 메트릭 조회 불가)
- 테넌트별로 개별 limits(레이트 리밋, retention 등) 설정 가능
- S3에서는 `<bucket>/<tenant-id>/` 경로로 분리 저장

```
Prometheus (팀 A) ──X-Scope-OrgID: team-a──▶ Mimir
Prometheus (팀 B) ──X-Scope-OrgID: team-b──▶ Mimir
                                                │
                                    ┌───────────┴──────────┐
                                 team-a               team-b
                         S3: .../team-a/       S3: .../team-b/
                            (격리된 블록)          (격리된 블록)
```

---

## 단일 테넌트 vs 멀티 테넌트

### 단일 테넌트 모드 (anonymous)

헤더 없이 사용하려면 `multitenancy_enabled: false` 설정:

```yaml
mimir:
  structuredConfig:
    multitenancy_enabled: false
```

이 경우 모든 데이터가 `anonymous` 테넌트로 저장됩니다.

> **주의**: `multitenancy_enabled`는 한 번 결정하면 변경이 어렵습니다.
> 처음부터 `true`로 시작하고 `default` 테넌트를 사용하는 것을 권장합니다.

### 멀티 테넌트 모드 (기본값)

```yaml
mimir:
  structuredConfig:
    multitenancy_enabled: true   # 기본값
```

---

## 테넌트 ID 규칙

- 영문, 숫자, `-`, `_`, `.` 사용 가능
- 대소문자 구분 (`team-a` ≠ `Team-A`)
- 공백 및 특수문자 불가 → `400 Bad Request`
- 최대 150자

**권장 네이밍 패턴:**
```
# 팀 기반
team-backend, team-frontend, team-data

# 환경 기반
prod, staging, dev

# 복합
prod-backend, staging-team-a
```

---

## Prometheus에서 테넌트 헤더 전송

```yaml
# prometheus.yaml
remote_write:
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: team-a
```

---

## Grafana에서 테넌트 선택

### 방법 1: 데이터소스별 고정 테넌트

```yaml
# grafana-datasource.yaml
datasources:
  - name: Mimir (team-a)
    type: prometheus
    url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
    jsonData:
      httpHeaderName1: X-Scope-OrgID
    secureJsonData:
      httpHeaderValue1: team-a

  - name: Mimir (team-b)
    type: prometheus
    url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
    jsonData:
      httpHeaderName1: X-Scope-OrgID
    secureJsonData:
      httpHeaderValue1: team-b
```

### 방법 2: 대시보드 변수로 동적 선택

Grafana 대시보드 Variables에서:
```
Name: tenant
Type: Custom
Values: team-a, team-b, team-c
```

대시보드 쿼리에서: `X-Scope-OrgID: ${tenant}`

---

## 테넌트별 Limits 설정

### 전역 기본값 (values.yaml)

```yaml
mimir:
  structuredConfig:
    limits:
      ingestion_rate: 10000           # 초당 최대 샘플 수
      ingestion_burst_size: 200000    # 버스트 허용량
      max_label_names_per_series: 30  # 시계열당 최대 레이블 수
      max_global_series_per_user: 1500000   # 테넌트당 최대 시계열 수
      compactor_blocks_retention_period: 30d
      max_query_parallelism: 32
      max_cache_freshness: 10m
```

### Runtime Config (테넌트별 오버라이드, 재시작 없이 적용)

```yaml
# runtime-config.yaml
overrides:
  team-a:
    ingestion_rate: 50000
    ingestion_burst_size: 1000000
    max_global_series_per_user: 5000000
    compactor_blocks_retention_period: 90d

  team-b:
    ingestion_rate: 5000
    max_global_series_per_user: 500000
    compactor_blocks_retention_period: 14d

  team-infra:
    # Mimir 자체 메타 모니터링용 테넌트 (제한 완화)
    ingestion_rate: 100000
    max_global_series_per_user: 10000000
    compactor_blocks_retention_period: 365d
```

#### Runtime Config를 ConfigMap으로 배포

```bash
# ConfigMap 생성
kubectl create configmap mimir-runtime-config \
  --from-file=runtime-config.yaml \
  -n monitoring

# Helm values에 마운트 설정
```

```yaml
# values.yaml
mimir:
  structuredConfig:
    runtime_config:
      file: /var/mimir/runtime-config.yaml
      period: 10s   # 변경 감지 주기

# ConfigMap 마운트
extraVolumes:
  - name: runtime-config
    configMap:
      name: mimir-runtime-config

extraVolumeMounts:
  - name: runtime-config
    mountPath: /var/mimir/runtime-config.yaml
    subPath: runtime-config.yaml
```

#### Runtime Config 실시간 업데이트

```bash
# ConfigMap 업데이트 (재배포 없이 적용)
kubectl create configmap mimir-runtime-config \
  --from-file=runtime-config.yaml \
  -n monitoring \
  --dry-run=client -o yaml | kubectl apply -f -

# 적용 확인 (10초 내 반영)
curl http://localhost:8080/runtime_config \
  -H "X-Scope-OrgID: team-a" | jq
```

---

## 테넌트 관리 API

```bash
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring

# 테넌트 목록 (S3 버킷 경로 기준)
aws s3 ls s3://mycompany-mimir-blocks/

# 테넌트의 레이블 목록 조회
curl -H "X-Scope-OrgID: team-a" \
  "http://localhost:8080/prometheus/api/v1/labels" | jq

# 테넌트의 메트릭 목록 조회
curl -H "X-Scope-OrgID: team-a" \
  "http://localhost:8080/prometheus/api/v1/label/__name__/values" | jq

# 테넌트의 현재 활성 시계열 수 확인
curl -H "X-Scope-OrgID: team-a" \
  "http://localhost:8080/prometheus/api/v1/query?query=count({__name__=~\".+\"})" | jq
```

---

## Cross-Tenant 쿼리 (주의: 실험적 기능)

관리자가 여러 테넌트를 동시에 쿼리해야 할 경우:

```bash
# 쉼표로 구분된 테넌트 ID (관리자용)
curl -H "X-Scope-OrgID: team-a|team-b" \
  "http://localhost:8080/prometheus/api/v1/query?query=up"
```

> **보안 주의**: `|` 구분 쿼리는 Nginx Gateway에서 차단하는 것이 좋습니다.
> 일반 사용자에게는 노출하지 마세요.

---

## 주의사항

- 한 번 데이터가 쓰인 테넌트 ID는 S3에 데이터가 남으므로 **이름 변경 불가**
- 테넌트 데이터를 완전히 삭제하려면 Compactor의 tenant deletion API 사용
- 테넌트 ID는 URL 경로에 포함되므로 URL 인코딩 문제 주의 (슬래시 사용 금지)

```bash
# 테넌트 데이터 삭제 (비가역적)
curl -X POST \
  "http://localhost:8080/compactor/delete_tenant" \
  -H "X-Scope-OrgID: old-tenant"
```
