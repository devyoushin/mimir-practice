# 트러블슈팅 가이드

Mimir 운영 중 자주 발생하는 문제와 해결 방법입니다.

---

## 빠른 진단 명령어

```bash
# 1. 전체 파드 상태 한눈에 보기
kubectl get pods -n monitoring -o wide \
  | awk '{print $1, $3, $4, $7}' | column -t

# 2. 재시작이 많은 파드 확인
kubectl get pods -n monitoring \
  --sort-by='.status.containerStatuses[0].restartCount' | tail -10

# 3. Mimir ready 상태
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring &
curl -s http://localhost:8080/ready

# 4. 링 상태 요약
for COMP in ingester distributor store-gateway compactor; do
  COUNT=$(curl -s http://localhost:8080/${COMP}/ring 2>/dev/null | jq '.shards | length' 2>/dev/null || echo "N/A")
  echo "${COMP}: ${COUNT} members in ring"
done
```

---

## 범주별 문제 해결

### 1. 설치 / 시작 문제

#### Ingester가 `CrashLoopBackOff` 또는 `Init:0/1`

```bash
# 로그 확인
kubectl logs -n monitoring mimir-ingester-0 --previous

# 주요 원인 1: S3 접근 실패
# 로그에서 "AccessDenied" 또는 "NoSuchBucket" 확인
kubectl logs -n monitoring mimir-ingester-0 | grep -i "s3\|access\|bucket"

# 해결: IRSA 설정 확인
kubectl get serviceaccount mimir -n monitoring -o yaml | grep role-arn
# eks.amazonaws.com/role-arn 어노테이션이 있어야 함

# IRSA 동작 확인
kubectl exec -n monitoring mimir-ingester-0 -- env | grep -E "AWS_ROLE_ARN|AWS_WEB_IDENTITY"
```

#### Store Gateway `Pending` 상태

```bash
# PVC 상태 확인
kubectl get pvc -n monitoring
# Pending이면 StorageClass 문제

# 사용 가능한 StorageClass 확인
kubectl get storageclass

# StorageClass 명시 (gp2 또는 gp3)
# values.yaml에 추가:
# store_gateway:
#   persistentVolume:
#     storageClass: gp2
```

---

### 2. 쓰기 경로 (Remote Write) 문제

#### `429 Too Many Requests`

Ingestion rate limit 초과:

```bash
# 현재 설정된 rate limit 확인
curl -s http://localhost:8080/runtime_config \
  -H "X-Scope-OrgID: default" | jq '.ingestion_rate'

# 현재 ingestion rate 확인
curl -s "http://localhost:8080/prometheus/api/v1/query?query=rate(cortex_distributor_received_samples_total[1m])" \
  -H "X-Scope-OrgID: default" | jq '.data.result[0].value[1]'
```

```yaml
# values.yaml에서 limit 증가
mimir:
  structuredConfig:
    limits:
      ingestion_rate: 50000        # 기본 10000에서 증가
      ingestion_burst_size: 1000000
```

또는 runtime-config.yaml에서 재시작 없이 조정.

#### Prometheus WAL 지연 급증

```bash
# WAL 지연 확인 (초)
curl -s "http://localhost:9090/api/v1/query?query=prometheus_remote_storage_highest_timestamp_in_seconds - prometheus_remote_storage_queue_highest_sent_timestamp_seconds" | jq '.data.result[0].value[1]'
# 정상: < 30초, 경보: > 300초

# 원인 1: max_shards 부족
# prometheus remote_write queue_config.max_shards를 200 → 50으로 줄이기 (네트워크 혼잡 시)
# 또는 max_shards를 늘려서 처리량 증가

# 원인 2: Distributor 병목
kubectl top pods -n monitoring -l app.kubernetes.io/component=distributor
# CPU가 높으면 Distributor 스케일 업
kubectl scale deployment mimir-distributor -n monitoring --replicas=3
```

#### `400 Bad Request` — 레이블 관련

```bash
# Distributor 로그에서 구체적 오류 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=distributor \
  | grep "400\|invalid\|label" | head -10

# 주요 원인:
# - 레이블 이름이 유효하지 않음 (영문, 숫자, _ 만 허용)
# - 레이블 수가 max_label_names_per_series(30) 초과
# - 메트릭 이름 형식 오류
```

---

### 3. 읽기 경로 (Query) 문제

#### Grafana에서 `No data` 또는 빈 그래프

```bash
# 원인 1: X-Scope-OrgID 불일치
# remote_write에서 사용한 tenant ID와 Grafana 데이터소스의 header 값이 동일해야 함
curl -s "http://localhost:8080/prometheus/api/v1/label/__name__/values" \
  -H "X-Scope-OrgID: default" | jq '.data | length'
# 0이면 해당 테넌트에 데이터 없음

# 원인 2: 시간 범위 문제
# Grafana 대시보드의 시간 범위가 데이터가 있는 기간을 포함하는지 확인

# 원인 3: Ingester가 아직 flush하지 않음
# 설치 후 2시간은 S3에 데이터가 없음 (Ingester 메모리에만 있음)
# Querier가 Ingester에서 직접 읽으므로 정상적으로 보여야 함
curl -s "http://localhost:8080/prometheus/api/v1/query?query=up" \
  -H "X-Scope-OrgID: default" | jq '.data.result | length'
```

#### 쿼리 타임아웃 (`context deadline exceeded`)

```bash
# 느린 쿼리 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=query-frontend \
  | grep "slow\|timeout\|deadline" | tail -20

# 해결책 1: 쿼리 범위 줄이기
# Grafana에서 시간 범위를 7d → 1d로 축소

# 해결책 2: Recording Rule로 미리 계산
# 자주 사용하는 무거운 쿼리를 Recording Rule로 대체

# 해결책 3: 타임아웃 증가
# values.yaml:
# mimir:
#   structuredConfig:
#     limits:
#       query_timeout: 120s
```

---

### 4. Compactor 문제

#### Compactor가 실행되지 않음

```bash
# Compactor 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=compactor \
  --since=1h | grep -i "error\|fail"

# 마지막 성공 실행 시각 확인
curl -s "http://localhost:8080/prometheus/api/v1/query?query=time()-cortex_compactor_last_successful_run_timestamp_seconds" \
  -H "X-Scope-OrgID: default" | jq '.data.result[0].value[1]'
# 단위: 초. 3600(1시간) 이상이면 문제

# Compactor PVC 용량 확인
kubectl exec -n monitoring mimir-compactor-0 -- df -h /data
```

---

### 5. Ruler / Alerting 문제

#### 알림이 발동하지 않음

```bash
# 규칙 평가 상태 확인
curl -s http://localhost:8080/prometheus/api/v1/rules \
  -H "X-Scope-OrgID: default" | \
  jq '.data.groups[].rules[] | {name: .name, health: .health, lastError: .lastError}'

# 오류 있는 규칙만
curl -s http://localhost:8080/prometheus/api/v1/rules \
  -H "X-Scope-OrgID: default" | \
  jq '.data.groups[].rules[] | select(.health != "ok")'

# Ruler 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=ruler \
  | grep -i "error\|eval" | tail -20

# 알림 상태 직접 확인
curl -s http://localhost:8080/prometheus/api/v1/alerts \
  -H "X-Scope-OrgID: default" | jq '.data.alerts[] | {name: .labels.alertname, state: .state}'
```

#### Alertmanager로 알림이 전달되지 않음

```bash
# Alertmanager 설정 확인
curl -s http://localhost:8080/api/v1/alerts \
  -H "X-Scope-OrgID: default" | jq

# Alertmanager 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=alertmanager \
  | grep -i "error\|webhook\|slack" | tail -20

# Alertmanager UI 확인
kubectl port-forward svc/mimir-alertmanager 9093:9093 -n monitoring
# http://localhost:9093
```

---

### 6. S3 / 스토리지 문제

#### Ingester가 S3에 flush하지 못함

```bash
# S3 오류 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=ingester \
  | grep -i "s3\|flush\|upload\|error" | tail -20

# S3 버킷에 블록이 생성되는지 확인
aws s3 ls s3://mycompany-mimir-blocks/ --recursive | wc -l
# 2시간 후에도 0이면 flush 실패

# IRSA 권한 재확인
kubectl exec -n monitoring mimir-ingester-0 -- \
  aws s3 ls s3://mycompany-mimir-blocks/ --region ap-northeast-2
```

---

## 로그 수집 명령어 모음

```bash
# 전체 컴포넌트 최근 에러 로그 수집
for COMP in distributor ingester querier query-frontend store-gateway compactor ruler alertmanager; do
  echo "===== ${COMP} errors ====="
  kubectl logs -n monitoring \
    -l app.kubernetes.io/component=${COMP} \
    --since=30m 2>/dev/null \
    | grep -i "error\|fail\|panic" | tail -5
done
```

---

## 유용한 진단 PromQL

```promql
# 샘플 수신 성공률
sum(rate(cortex_distributor_received_samples_total[5m]))
  /
sum(rate(cortex_distributor_samples_in_total[5m]))

# Ingester 메모리 사용량 (시계열)
cortex_ingester_memory_series

# Query Frontend 캐시 히트율
sum(rate(cortex_cache_hits_total[5m]))
  /
sum(rate(cortex_cache_fetched_total[5m]))

# S3 요청 오류율
sum(rate(thanos_objstore_bucket_operation_failures_total[5m]))
  /
sum(rate(thanos_objstore_bucket_operations_total[5m]))

# Compactor 미처리 블록 수
cortex_bucket_blocks_count - cortex_bucket_blocks_marked_for_deletion_total
```

---

## 긴급 상황 대응

### Ingester 데이터 손실 우려 시

```bash
# 즉시 S3로 flush 유도 (파드 graceful 종료)
kubectl delete pod mimir-ingester-0 -n monitoring --grace-period=60
# Ingester는 종료 전 자동으로 S3 flush를 시도합니다
```

### 링이 완전히 깨진 경우 (모든 Ingester 비정상)

```bash
# 링 KV 상태 초기화 (주의: 데이터 유실 가능)
kubectl exec -n monitoring mimir-distributor-xxx -- \
  curl -X DELETE http://localhost:8080/ingester/ring
# 이후 모든 Ingester 재시작
kubectl rollout restart statefulset/mimir-ingester -n monitoring
```

> **항상 Grafana On-Call 또는 PagerDuty로 에스컬레이션하기 전에**
> Mimir 컴포넌트의 로그와 ring 상태를 먼저 확인하세요.
