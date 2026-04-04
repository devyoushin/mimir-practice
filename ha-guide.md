# 고가용성 (Zone-aware Replication)

Mimir는 기본적으로 `replication_factor=3`으로 데이터를 복제합니다.
Zone-aware replication을 활성화하면 **가용 영역(AZ) 전체 장애에도** 데이터를 보호합니다.

---

## Replication Factor 이해

Mimir는 Distributor → Ingester 전송 시 데이터를 `replication_factor`만큼 복제합니다.

```
Distributor
    │ replication_factor=3
    ├──▶ Ingester-0 (AZ-a)
    ├──▶ Ingester-1 (AZ-b)
    └──▶ Ingester-2 (AZ-c)
```

- **Write Quorum**: `ceil(3 / 2) = 2`개 이상 성공해야 수락
- **Read Quorum**: 최소 1개에서 읽으면 되지만 중복 제거 후 반환
- **내결함성**: 최대 1개 Ingester 장애 허용

---

## Zone-aware Replication

### 문제: Zone-unaware replication의 위험성

```
Zone-unaware (기본값 문제):
  Ingester-0 (AZ-a)  ← 복제본 1
  Ingester-1 (AZ-a)  ← 복제본 2    ← AZ-a 전체 장애 시 2/3 복제본 소실!
  Ingester-2 (AZ-b)  ← 복제본 3

Zone-aware replication:
  Ingester-0 (AZ-a)  ← 복제본 1    ← AZ 1개 장애 시 항상 2/3 복제본 안전
  Ingester-1 (AZ-b)  ← 복제본 2
  Ingester-2 (AZ-c)  ← 복제본 3
```

---

## Zone-aware Replication 설정 (values-ha.yaml)

### 1. Ingester — Zone-aware

```yaml
# helm/values-ha.yaml
ingester:
  zoneAwareReplication:
    enabled: true
    zones:
      - name: zone-a
        nodeSelector:
          topology.kubernetes.io/zone: ap-northeast-2a
      - name: zone-b
        nodeSelector:
          topology.kubernetes.io/zone: ap-northeast-2b
      - name: zone-c
        nodeSelector:
          topology.kubernetes.io/zone: ap-northeast-2c
  replicas: 2    # AZ당 2개 → 총 6개 (고부하 환경)
  # replicas: 1  # AZ당 1개 → 총 3개 (기본 HA)
```

### 2. Store Gateway — Zone-aware

```yaml
store_gateway:
  zoneAwareReplication:
    enabled: true
    zones:
      - name: zone-a
        nodeSelector:
          topology.kubernetes.io/zone: ap-northeast-2a
      - name: zone-b
        nodeSelector:
          topology.kubernetes.io/zone: ap-northeast-2b
      - name: zone-c
        nodeSelector:
          topology.kubernetes.io/zone: ap-northeast-2c
  replicas: 1    # AZ당 1개 → 총 3개
```

### 3. Mimir structuredConfig

```yaml
mimir:
  structuredConfig:
    ingester:
      ring:
        zone_awareness_enabled: true
        replication_factor: 3

    store_gateway:
      sharding_ring:
        zone_awareness_enabled: true
        replication_factor: 3
```

---

## 컴포넌트별 HA 권장 구성

| 컴포넌트 | 유형 | 최소 (개발) | 기본 HA | 고가용 HA |
|---|---|---|---|---|
| Distributor | Stateless | 1 | 3+ | AZ별 2+ |
| Ingester | StatefulSet | 3 (RF=3) | AZ당 1 (총 3) | AZ당 2 (총 6) |
| Query Frontend | Stateless | 1 | 2+ | 3+ |
| Querier | Stateless | 2 | 3+ | AZ별 1+ |
| Store Gateway | StatefulSet | 1 | AZ당 1 (총 3) | AZ당 2 (총 6) |
| Compactor | StatefulSet | 1 | 1 | 2 (sharding 필요) |
| Ruler | Stateless | 1 | 2+ | 3+ |
| Alertmanager | StatefulSet | 1 | 3 (RF=3) | 3 |

---

## PodDisruptionBudget

클러스터 유지보수(노드 drain) 시 Quorum 깨지는 것을 방지:

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mimir-ingester-pdb
  namespace: monitoring
spec:
  maxUnavailable: 1   # 한 번에 1개까지만 중단 허용
  selector:
    matchLabels:
      app.kubernetes.io/component: ingester
      app.kubernetes.io/instance: mimir
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mimir-store-gateway-pdb
  namespace: monitoring
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: store-gateway
      app.kubernetes.io/instance: mimir
```

```bash
kubectl apply -f pdb.yaml
kubectl get pdb -n monitoring
```

---

## Topology Spread Constraints

Distributor, Querier 등 Stateless 컴포넌트를 AZ에 균등 분산:

```yaml
distributor:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/component: distributor
```

---

## Ingester 롤링 재시작 (안전하게)

Ingester StatefulSet 재시작 시 인메모리 데이터 손실 방지:

```yaml
mimir:
  structuredConfig:
    ingester:
      ring:
        # 종료 시 담당 토큰을 다른 Ingester에 이전
        unregister_on_shutdown: true
        # Handoff 완료까지 대기
        final_sleep: 0s
```

**수동 롤링 재시작 (권장):**

```bash
# StatefulSet을 한 번에 1개씩 재시작
for i in 0 1 2; do
  echo "=== Restarting ingester-${i} ==="
  kubectl delete pod mimir-ingester-${i} -n monitoring

  # Running 상태가 될 때까지 대기
  kubectl wait pod mimir-ingester-${i} -n monitoring \
    --for=condition=Ready \
    --timeout=5m

  echo "=== ingester-${i} Ready ==="
  sleep 30   # 안정화 대기
done
```

**또는 Helm의 maxUnavailable 설정:**

```yaml
ingester:
  statefulSet:
    updateStrategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 1   # 동시 재시작 최대 1개
```

---

## Alertmanager HA 설정

```yaml
mimir:
  structuredConfig:
    alertmanager:
      enable_api: true
      sharding_enabled: true
      sharding_ring:
        replication_factor: 3

alertmanager:
  replicas: 3
  persistentVolume:
    enabled: true
    size: 5Gi
```

---

## 장애 시나리오 테스트

### Ingester 1개 장애 테스트

```bash
# Ingester 1개 강제 종료 (쓰기/읽기가 중단되지 않아야 함)
kubectl delete pod mimir-ingester-1 -n monitoring

# 쓰기 에러율 확인 (0이어야 함)
curl "http://localhost:8080/prometheus/api/v1/query?query=rate(cortex_distributor_ingestion_failures_total[5m])" \
  -H "X-Scope-OrgID: default" | jq '.data.result'

# 파드가 자동 재시작되는지 확인
kubectl get pods -n monitoring -l app.kubernetes.io/component=ingester -w
```

### AZ 장애 시뮬레이션

```bash
# AZ-a 노드 격리
kubectl cordon <node-in-az-a>
kubectl drain <node-in-az-a> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30

# Ingester 링 상태 확인 (일부 LEAVING 상태 정상)
curl http://localhost:8080/ingester/ring | jq

# 쓰기/읽기 계속 동작하는지 확인
curl "http://localhost:8080/prometheus/api/v1/query?query=up" \
  -H "X-Scope-OrgID: default" | jq '.status'

# 복구 후 노드 uncordon
kubectl uncordon <node-in-az-a>
```

---

## HA 상태 점검 체크리스트

```bash
# 1. Ingester 링 상태 (모두 ACTIVE여야 함)
curl http://localhost:8080/ingester/ring | jq '.shards[] | {id: .id, state: .state, zone: .zone}'

# 2. Store Gateway 링 상태
curl http://localhost:8080/store-gateway/ring | jq '.shards[] | {id: .id, state: .state}'

# 3. PDB 상태 확인
kubectl get pdb -n monitoring

# 4. Pod 분산 확인 (AZ별 분포)
kubectl get pods -n monitoring \
  -l app.kubernetes.io/component=ingester \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName' | \
  sort

# 5. replication_factor 적용 확인
curl "http://localhost:8080/prometheus/api/v1/query?query=cortex_ring_members" \
  -H "X-Scope-OrgID: default" | jq
```

---

## 참고 링크

- [Zone-aware replication 공식 문서](https://grafana.com/docs/mimir/latest/operators-guide/configure/configure-zone-aware-replication/)
- [Mimir HA 배포 가이드](https://grafana.com/docs/mimir/latest/operators-guide/deploy-grafana-mimir/)
- [PodDisruptionBudget 가이드](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
