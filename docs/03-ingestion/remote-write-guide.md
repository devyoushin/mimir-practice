# Write Path — Prometheus Remote Write

Prometheus에서 Mimir로 메트릭을 전송하는 방법입니다.

---

## Remote Write 개요

```
Prometheus (또는 Prometheus Agent)
    │  HTTP POST, Snappy 압축 protobuf
    │  X-Scope-OrgID: <tenant>
    ▼
mimir-nginx (Gateway)
    │
    ▼
Distributor
    │ consistent hashing
    │ replication_factor=3
    ▼
Ingester × 3 (AZ별 1개)
    │
    ▼ (2시간마다)
S3 (TSDB 블록)
```

---

## Prometheus 설정

### 기본 설정

```yaml
# prometheus.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  # 모든 메트릭에 클러스터 레이블 추가 (권장)
  external_labels:
    cluster: my-eks-cluster
    environment: production

remote_write:
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: default
    queue_config:
      capacity: 2500
      max_samples_per_send: 10000
      max_shards: 200
      min_shards: 1
      batch_send_deadline: 5s
      min_backoff: 30ms
      max_backoff: 5s
    remote_timeout: 30s
```

### Prometheus Agent 모드 (경량화, 권장)

Prometheus를 long-term storage 없이 **scrape + remote_write 전용**으로 사용할 경우:

```bash
# Agent 모드 활성화 (로컬 스토리지 불필요)
prometheus --enable-feature=agent
```

```yaml
# prometheus-agent.yaml
global:
  scrape_interval: 15s
  external_labels:
    cluster: my-eks-cluster

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

remote_write:
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: default
    queue_config:
      max_samples_per_send: 10000
      max_shards: 50
```

> Agent 모드는 WAL만 로컬에 유지하고, 장기 저장은 Mimir에 위임합니다.
> 메모리 사용량이 크게 줄어듭니다.

---

## Queue Config 튜닝

| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `capacity` | 2500 | 큐 버퍼 크기 (샘플 수) |
| `max_shards` | 200 | 최대 병렬 전송 goroutine 수 |
| `min_shards` | 1 | 최소 병렬 전송 수 |
| `max_samples_per_send` | 500 | 한 번의 HTTP 요청에 담는 최대 샘플 수 |
| `batch_send_deadline` | 5s | max_samples_per_send에 도달하지 않아도 전송하는 최대 대기 시간 |
| `min_backoff` | 30ms | 재시도 최소 대기 |
| `max_backoff` | 5s | 재시도 최대 대기 |

### 환경별 튜닝 가이드

**소규모 (시계열 < 10만개):**
```yaml
queue_config:
  capacity: 2500
  max_shards: 50
  max_samples_per_send: 5000
```

**대규모 (시계열 > 100만개):**
```yaml
queue_config:
  capacity: 10000
  max_shards: 200
  max_samples_per_send: 10000
  batch_send_deadline: 5s
```

**네트워크 불안정 환경:**
```yaml
queue_config:
  max_shards: 20          # 동시 요청 줄이기
  min_backoff: 100ms
  max_backoff: 30s        # 더 오래 기다리기
```

---

## 전송 상태 모니터링

Prometheus 자체 메트릭으로 remote write 상태를 확인합니다:

```promql
# 전송 대기 중인 샘플 수 (이 값이 크면 병목)
prometheus_remote_storage_samples_pending

# 초당 전송 성공 샘플 수
rate(prometheus_remote_storage_samples_total[5m])

# 초당 전송 실패 수 (0이어야 정상)
rate(prometheus_remote_storage_samples_failed_total[5m])

# WAL 지연 = 스크랩 시각 vs 실제 전송 시각의 차이 (초)
# 이 값이 크면 전송 병목
prometheus_remote_storage_highest_timestamp_in_seconds
  - prometheus_remote_storage_queue_highest_sent_timestamp_seconds

# 활성 shards 수 (max_shards에 가까우면 병목 신호)
prometheus_remote_storage_shards
```

### Remote Write 상태 확인 대시보드

Grafana에서 Prometheus 자체를 scrape하면 위 메트릭을 볼 수 있습니다.
또는 Prometheus UI (`http://localhost:9090/targets`) 에서 WAL 상태 확인.

---

## 여러 테넌트로 분리 전송

```yaml
remote_write:
  # team-a 네임스페이스의 메트릭 → team-a 테넌트
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: team-a
    write_relabel_configs:
      - source_labels: [namespace]
        regex: "team-a(-.*)?"
        action: keep

  # team-b 네임스페이스의 메트릭 → team-b 테넌트
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: team-b
    write_relabel_configs:
      - source_labels: [namespace]
        regex: "team-b(-.*)?"
        action: keep

  # 전체를 default 테넌트에도 복제 (옵션)
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: default
```

---

## 레이블 필터링 (write_relabel_configs)

```yaml
remote_write:
  - url: http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push
    headers:
      X-Scope-OrgID: default
    write_relabel_configs:
      # 내부 메트릭 제외 (go_, process_ 등)
      - source_labels: [__name__]
        regex: "(go|process)_.*"
        action: drop

      # 불필요한 레이블 제거
      - action: labeldrop
        regex: "pod_template_hash"

      # 특정 job만 전송
      - source_labels: [job]
        regex: "my-service|another-service"
        action: keep
```

---

## 전송 테스트

```bash
# Mimir Gateway 포트 포워드
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring

# 수신 확인 (instant query)
curl "http://localhost:8080/prometheus/api/v1/query?query=up" \
  -H "X-Scope-OrgID: default" | jq '.data.result | length'

# Distributor 통계 확인
curl http://localhost:8080/distributor/all_user_stats \
  -H "X-Scope-OrgID: default" | jq

# 실시간 ingest rate 확인 (PromQL)
curl "http://localhost:8080/prometheus/api/v1/query?query=sum(rate(cortex_ingester_ingested_samples_total[1m]))" \
  -H "X-Scope-OrgID: default" | jq '.data.result[0].value[1]'
```

---

## Kubernetes에서 Prometheus 배포 (Helm)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.global.external_labels.cluster=my-eks-cluster \
  --set "server.remoteWrite[0].url=http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push" \
  --set "server.remoteWrite[0].headers.X-Scope-OrgID=default"
```

또는 kube-prometheus-stack 사용 (Grafana + 알림 규칙 포함):

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.remoteWrite[0].url="http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push" \
  --set "prometheus.prometheusSpec.remoteWrite[0].headers.X-Scope-OrgID=default"
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| `429 Too Many Requests` | `ingestion_rate` 초과 | Mimir limits.ingestion_rate 증가 |
| `400 Bad Request` | 잘못된 레이블 이름 또는 테넌트 ID | 레이블 이름 확인, X-Scope-OrgID 확인 |
| `500 Internal Server Error` | Ingester 장애 | `kubectl get pods -n monitoring` 로 상태 확인 |
| WAL 지연 급증 | 네트워크/Distributor 병목 | `max_shards` 감소, Distributor 스케일 업 |
| `context deadline exceeded` | remote_write 타임아웃 | `remote_timeout: 60s` 로 증가 |
| `max shards exceeded` | Mimir 처리 불가 | Distributor 증설 또는 `max_samples_per_send` 축소 |

```bash
# Prometheus remote_write 에러 로그 확인
kubectl logs -n monitoring -l app=prometheus -c prometheus \
  | grep -i "remote_write\|failed\|error" | tail -20

# Distributor 수신 오류 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=distributor \
  | grep -i "error\|429\|400" | tail -20
```
