# 모니터링 규칙 — mimir-practice

## Mimir 자체 모니터링

### 핵심 메트릭

```promql
# Ingester 활성 시계열 수
sum(cortex_ingester_memory_series)

# 수집 속도 (샘플/초)
sum(rate(cortex_distributor_received_samples_total[5m]))

# 수집 제한 초과 횟수
sum(rate(cortex_discarded_samples_total[5m])) by (reason)

# 쿼리 지연 (p99)
histogram_quantile(0.99, sum(rate(cortex_request_duration_seconds_bucket{route=~"api_v1_query.*"}[5m])) by (le))

# Compactor 마지막 성공 시각
cortex_compactor_last_successful_run_timestamp_seconds
```

### 알림 규칙 (권장)

| 알림 | 조건 | 심각도 |
|------|------|--------|
| MimirIngesterDown | Ingester 다운 | critical |
| MimirIngestionRateLimitExceeded | 제한 초과 > 0 | warning |
| MimirCompactorNotRunning | 마지막 실행 > 24h | warning |
| MimirHighCardinality | 시계열 수 임계 근접 | warning |

## SLO 정의 (Mimir 서비스)

| SLI | 목표 | 측정 방법 |
|-----|------|---------|
| 수집 가용성 | 99.9% | 수집 오류율 < 0.1% |
| 쿼리 p99 지연 | < 10초 | `cortex_request_duration_seconds` |
| 데이터 보존 | 설정 기간 100% | Compactor 성공률 |

## 대시보드 구성 (Grafana)
- **Overview**: 수집 속도, 시계열 수, 쿼리 수
- **Distributor**: 수집 제한 현황, 샤딩 상태
- **Ingester**: 메모리 사용량, 청크 상태, 링 상태
- **Compactor**: 블록 처리 현황, 실행 스케줄
- **Per-Tenant**: 테넌트별 사용량 분석
