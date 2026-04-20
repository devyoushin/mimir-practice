# Mimir 컨벤션 규칙

## 테넌트 네이밍
- 형식: `{team}-{env}` 또는 `{service}-{env}`
- 예: `platform-prod`, `payment-stg`, `infra-dev`
- 소문자, 하이픈(-) 허용, 특수문자 금지
- 최대 150자

## Per-Tenant 제한 설정 구조
```yaml
# mimir-config.yaml의 limits 섹션
limits:
  # 기본값 (모든 테넌트)
  ingestion_rate: 10000
  ingestion_burst_size: 20000
  max_global_series_per_user: 1000000
  max_label_names_per_series: 30

overrides:
  # 팀별 오버라이드
  platform-prod:
    ingestion_rate: 100000
    max_global_series_per_user: 10000000
```

## 메트릭 카디널리티 제한
- 시계열 수 상한: 테넌트당 1M (기본), 10M (대형 테넌트)
- 레이블 수: 시계열당 최대 30개
- 레이블 값 길이: 최대 1024자
- 고카디널리티 레이블 모니터링 필수

## remote_write 설정 권장
```yaml
remote_write:
  - url: http://mimir-distributor:8080/api/v1/push
    headers:
      X-Scope-OrgID: my-tenant
    queue_config:
      max_samples_per_send: 2000
      max_shards: 200
      min_backoff: 30ms
      max_backoff: 5s
```

## Compactor 스케줄 규칙
- 오프피크 시간대 실행 (02:00-06:00 KST)
- 보존 기간 종료 블록 자동 삭제
- retention_period: 테넌트별로 설정 가능

## 알림 규칙 컨벤션 (vmagent/Ruler)
- 그룹명: `{service}.{category}.rules`
- 알림명: PascalCase
- severity 레이블 필수: critical, warning, info
- runbook_url 어노테이션 필수
