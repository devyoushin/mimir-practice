# Compactor

Compactor는 S3에 저장된 TSDB 블록을 주기적으로 병합(compaction)하여
저장 공간을 최적화하고 쿼리 성능을 향상시킵니다.

---

## 왜 Compaction이 필요한가?

Ingester는 2시간마다 소규모 블록을 S3에 플러시합니다.
시간이 지날수록 작은 블록이 누적되면:

- 쿼리 시 수천 개의 블록을 스캔해야 함 → **쿼리 속도 저하**
- 블록 메타데이터 오버헤드 증가 → **Store Gateway 메모리 부하**
- S3 LIST/GET 요청 증가 → **비용 증가**

Compactor가 이 블록들을 병합합니다:

```
Before (compaction 전):                  After (compaction 후):
[2h] [2h] [2h] [2h] [2h] [2h]           [12h 블록]
[2h] [2h] [2h] [2h] [2h] [2h]   →       [12h 블록]
→ 12개 블록 → 쿼리 시 12번 스캔          → 2개 블록 → 쿼리 시 2번 스캔
```

---

## Compaction 레벨 (Block Ranges)

| 레벨 | 블록 범위 | 설명 |
|---|---|---|
| Level 1 | 0 ~ 2h | Ingester가 플러시한 원본 블록 |
| Level 2 | 2h ~ 12h | Level 1 블록 6개 병합 |
| Level 3 | 12h ~ 24h | Level 2 블록 2개 병합 |
| Level 4 | 24h ~ 48h | Level 3 블록 2개 병합 |

`block_ranges: [2h, 12h, 24h, 48h]` 설정 시의 동작:

```
Day 1:  [2h][2h][2h][2h][2h][2h] → 병합 → [12h][12h]
Day 2:  [12h][12h] → 병합 → [24h]
Day 3+: [24h][24h] → 병합 → [48h]
Day N:  retention 초과 → tombstone → 삭제
```

---

## 설정

```yaml
# helm/values.yaml
mimir:
  structuredConfig:
    compactor:
      # 블록 범위 (계층적 병합 단계)
      block_ranges:
        - 2h
        - 12h
        - 24h
        - 48h

      # 데이터 보존 기간 (0 = 무기한 → 비용 주의)
      retention_period: 90d

      # 컴팩션 상태 체크 주기
      cleanup_interval: 15m

      # 동시 블록 처리 수 (CPU/메모리와 트레이드오프)
      max_opening_blocks_concurrency: 4
      max_closing_blocks_concurrency: 2
      max_compaction_workers: 4

      # 여러 Compactor 파드 사용 시 (sharding)
      sharding_enabled: true
      ring:
        kvstore:
          store: memberlist

    limits:
      # 테넌트별 보존 기간 오버라이드 가능
      compactor_blocks_retention_period: 90d
```

---

## Compactor 운영 패턴

### 싱글톤 (기본, 소규모)

```yaml
compactor:
  replicas: 1   # 단일 파드로 충분
```

### Sharding (대규모, 다중 테넌트)

테넌트가 많아지면 Compactor를 수평 확장합니다:

```yaml
compactor:
  replicas: 2

mimir:
  structuredConfig:
    compactor:
      sharding_enabled: true
      ring:
        kvstore:
          store: memberlist
```

각 Compactor 파드는 링에서 할당된 테넌트만 담당합니다.

---

## Compactor 상태 확인

```bash
# Compactor 파드에 직접 접근
kubectl port-forward svc/mimir-compactor 8080:8080 -n monitoring

# Compactor 링 상태 (여러 파드 운영 시)
curl http://localhost:8080/compactor/ring | jq

# 현재 처리 중인 테넌트 목록
curl http://localhost:8080/compactor/tenants | jq
```

### 핵심 모니터링 메트릭 (PromQL)

```promql
# 마지막 성공적 컴팩션 이후 경과 시간 (1시간 이상이면 이상)
time() - cortex_compactor_last_successful_run_timestamp_seconds

# 컴팩션 실패 수 증가율
increase(cortex_compactor_runs_failed_total[1h])

# 삭제 대상으로 마킹된 블록 수 (이 값이 줄어야 정상)
cortex_compactor_blocks_marked_for_deletion_total

# 컴팩션에 소요된 시간 (P99)
histogram_quantile(0.99, rate(cortex_compactor_block_compaction_duration_seconds_bucket[1h]))

# S3에 있는 블록 총 수 (테넌트별)
cortex_bucket_blocks_count
```

---

## Compactor가 자동으로 하는 작업

1. **Compaction**: 소규모 블록 → 대형 블록 병합
2. **Deletion**: `retention_period` 초과 블록 → tombstone 마킹 → 실제 삭제
3. **Bucket Index 갱신**: Store Gateway가 참조하는 버킷 인덱스 업데이트

### 블록 삭제 절차 (안전 삭제)

```
블록 → tombstone 마킹 (DeletionMark)
    ↓ cleanup_interval × 2 (기본 30분) 대기
    ↓ Store Gateway 캐시 갱신 확인
    ↓ 실제 S3 삭제
```

직접 S3에서 블록을 삭제하면 메타데이터 불일치가 발생합니다. **절대 권장하지 않습니다.**

---

## 수동 개입이 필요한 경우

### 특정 테넌트 데이터 삭제

```bash
# 테넌트 데이터 삭제 API
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring

curl -X DELETE \
  http://localhost:8080/compactor/delete_tenant_status \
  -H "X-Scope-OrgID: tenant-to-delete"

# 삭제 상태 확인
curl http://localhost:8080/compactor/delete_tenant_status \
  -H "X-Scope-OrgID: tenant-to-delete" | jq
```

### Compactor 강제 재실행 (비상 시)

```bash
# Compactor 파드 재시작 (다음 cleanup_interval에 자동 재실행)
kubectl rollout restart statefulset/mimir-compactor -n monitoring

# 로그 모니터링
kubectl logs -n monitoring -l app.kubernetes.io/component=compactor -f
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| `cortex_compactor_last_successful_run` 갱신 안됨 | Compactor 파드 오류 | `kubectl logs` 로 오류 확인 |
| S3 블록 수가 계속 증가 | Compactor 중단 또는 설정 오류 | `block_ranges` 및 `sharding_enabled` 확인 |
| OOMKilled | 너무 많은 동시 컴팩션 | `max_opening_blocks_concurrency` 감소 |
| 컴팩션이 너무 오래 걸림 | PVC 성능 부족 | `storageClass: gp3` 로 변경 (gp2 대비 3x IOPS) |

```bash
# Compactor 오류 로그 확인
kubectl logs -n monitoring \
  -l app.kubernetes.io/component=compactor \
  --since=1h \
  | grep -i "error\|fail\|panic"
```

---

## 참고 링크

- [Mimir Compactor 공식 문서](https://grafana.com/docs/mimir/latest/operators-guide/architecture/components/compactor/)
- [Block Compaction 상세 설명](https://grafana.com/docs/mimir/latest/operators-guide/architecture/compactor/)
