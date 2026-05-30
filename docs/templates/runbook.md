# [알림명] — Mimir 런북

> 심각도: Critical / Warning / Info
> 담당팀: | 최종 수정: YYYY-MM-DD

## 알림 요약
<!-- 이 알림이 무엇을 감지하는지 한 문장으로 -->

## 알림 조건

```promql
# 알림 트리거 PromQL/MetricsQL
```

## 영향도
<!-- 이 알림이 발생했을 때 사용자/서비스에 미치는 영향 -->

## 즉시 확인 사항

```bash
# 1. Mimir 컴포넌트 상태 확인
kubectl get pods -n mimir

# 2. Ingester 링 상태
curl http://mimir-ingester:8080/ring

# 3. Compactor 상태
curl http://mimir-compactor:8080/compactor/ring
```

## 진단 단계

### 1단계: 컴포넌트 상태 파악
```bash
# Distributor 수집 오류 확인
kubectl logs -n mimir -l app=mimir-distributor --tail=50
```

### 2단계: 원인 분석
<!-- 일반적인 원인 목록 -->

### 3단계: 해결 조치
<!-- 단계별 해결 방법 -->

## 에스컬레이션
- 15분 내 해결 불가 시: 팀 리드 호출
- 메트릭 수집 전면 중단 시: 인시던트 선언

## 참고
- 관련 Grafana 대시보드:
- 관련 문서:
