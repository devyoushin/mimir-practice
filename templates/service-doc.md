# [서비스명] — Mimir 메트릭 저장소 문서

> 작성일: YYYY-MM-DD | 작성자: | 검토자:

## 개요
<!-- 이 Mimir 구성/테넌트의 목적과 역할 -->

## 구성 정보

| 항목 | 값 |
|------|-----|
| Mimir 버전 | |
| 배포 모드 | 단일 바이너리 / 마이크로서비스 |
| 테넌트 ID | |
| 스토리지 | S3 / GCS |
| 보존 기간 | |

## 수집 메트릭 현황

| 항목 | 값 |
|------|-----|
| Active Series | |
| 수집 속도 (samples/sec) | |
| 일일 스토리지 사용량 | |

## 주요 PromQL / MetricsQL 쿼리

```promql
# 주요 쿼리 예시
```

## 테넌트 제한 설정

```yaml
# per-tenant overrides
overrides:
  <tenant-id>:
    ingestion_rate: 10000
    max_global_series_per_user: 1000000
```

## 운영 체크리스트
- [ ] Ingester 링 상태 정상
- [ ] Compactor 최근 실행 성공
- [ ] 스토리지 연결 상태 확인
- [ ] 카디널리티 임계값 이하

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| | | |

## 관련 문서
- 런북:
- Grafana Mimir 대시보드:
- 테넌트 설정:
