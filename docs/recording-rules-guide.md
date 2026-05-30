# Recording Rules

Recording Rules는 복잡하거나 비용이 큰 PromQL 쿼리를 주기적으로 실행하여
결과를 새로운 메트릭으로 저장합니다.

---

## 왜 Recording Rules를 사용하나?

| 상황 | 문제 | 해결 |
|---|---|---|
| 복잡한 집계 쿼리 | Grafana 대시보드가 느림 | 미리 계산하여 저장 |
| 고카디널리티 데이터 | 쿼리마다 수백만 시계열 스캔 | 집계된 메트릭 사용 |
| Alerting Rules의 공통 조건 | 동일 쿼리 중복 실행 | Recording Rule로 공통화 |
| 대시보드 패널 20개가 동일 쿼리 | Querier 부하 급증 | 1개의 Rule로 통합 |

---

## 1. 규칙 파일 작성

### 네이밍 컨벤션

```
level:metric_name:operations
```

| 레벨 | 집계 대상 | 예시 |
|---|---|---|
| `instance` | 인스턴스 원본 | (Recording Rule 대상 아님) |
| `job` | job 레이블 기준 집계 | `job:http_requests:rate5m` |
| `namespace` | 네임스페이스 기준 집계 | `namespace:cpu_usage:ratio` |
| `cluster` | 클러스터 전체 집계 | `cluster:error_rate:ratio5m` |

### 실용적인 예시

```yaml
# recording-rules.yaml
groups:
  # ─────────────────────────────────────────────
  # HTTP 메트릭 (RED: Rate, Errors, Duration)
  # ─────────────────────────────────────────────
  - name: http:red
    interval: 1m
    rules:
      # 요청 비율 (job별)
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # 에러율 (job별)
      - record: job:http_errors:rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))

      # P50 레이턴시
      - record: job:http_request_duration_p50:rate5m
        expr: |
          histogram_quantile(0.50,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )

      # P99 레이턴시
      - record: job:http_request_duration_p99:rate5m
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )

  # ─────────────────────────────────────────────
  # Kubernetes 리소스
  # ─────────────────────────────────────────────
  - name: kubernetes:resources
    interval: 1m
    rules:
      # 네임스페이스별 CPU 사용량
      - record: namespace:container_cpu_usage:rate5m
        expr: |
          sum by (namespace) (
            rate(container_cpu_usage_seconds_total{container!=""}[5m])
          )

      # 네임스페이스별 메모리 사용량
      - record: namespace:container_memory_working_set:sum
        expr: |
          sum by (namespace) (
            container_memory_working_set_bytes{container!=""}
          )

      # 파드별 CPU 사용률 (limit 대비)
      - record: namespace_pod:container_cpu_usage_ratio:rate5m
        expr: |
          sum by (namespace, pod) (
            rate(container_cpu_usage_seconds_total{container!=""}[5m])
          )
          /
          sum by (namespace, pod) (
            container_spec_cpu_quota{container!=""} / container_spec_cpu_period{container!=""}
          )

  # ─────────────────────────────────────────────
  # 계층적 집계 (instance → job → cluster)
  # ─────────────────────────────────────────────
  - name: aggregation:hierarchy
    interval: 1m
    rules:
      # 1단계: 인스턴스 → job 집계
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      # 2단계: job → 전체 집계 (앞 단계 결과 재활용)
      - record: cluster:http_requests:rate5m
        expr: sum(job:http_requests:rate5m)

      # 클러스터 전체 에러율
      - record: cluster:http_errors:rate5m
        expr: |
          sum(job:http_errors:rate5m * job:http_requests:rate5m)
          /
          sum(job:http_requests:rate5m)
```

---

## 2. Ruler API로 업로드

```bash
# Mimir Gateway 포트 포워드
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring

# 규칙 업로드
# URL 형식: /prometheus/config/v1/rules/<namespace>
# namespace: Mimir 내부 규칙 네임스페이스 (Kubernetes 네임스페이스와 무관)
curl -X POST \
  http://localhost:8080/prometheus/config/v1/rules/my-app \
  -H "X-Scope-OrgID: default" \
  -H "Content-Type: application/yaml" \
  --data-binary @recording-rules.yaml

# 성공 시: HTTP 202 Accepted
```

---

## 3. 업로드 확인

```bash
# 등록된 모든 네임스페이스 + 규칙 조회
curl http://localhost:8080/prometheus/config/v1/rules \
  -H "X-Scope-OrgID: default" | jq

# 특정 네임스페이스의 규칙 조회
curl http://localhost:8080/prometheus/config/v1/rules/my-app \
  -H "X-Scope-OrgID: default" | jq

# 규칙 평가 상태 확인 (lastEvaluationTime, evaluationDuration 포함)
curl http://localhost:8080/prometheus/api/v1/rules \
  -H "X-Scope-OrgID: default" | jq '.data.groups[].rules[] | {name: .name, health: .health, lastEvaluation: .lastEvaluation}'

# 오류 있는 규칙만 필터
curl http://localhost:8080/prometheus/api/v1/rules \
  -H "X-Scope-OrgID: default" | jq '.data.groups[].rules[] | select(.health != "ok")'
```

---

## 4. mimirtool로 규칙 관리 (권장)

```bash
# mimirtool 설치 (Linux/Mac)
curl -Lo mimirtool https://github.com/grafana/mimir/releases/latest/download/mimirtool-linux-amd64
chmod +x mimirtool
sudo mv mimirtool /usr/local/bin/

# 또는 Go로 설치
go install github.com/grafana/mimir/pkg/mimirtool@latest

# 환경변수 설정
export MIMIR_ADDRESS=http://localhost:8080
export MIMIR_TENANT_ID=default

# 규칙 업로드
mimirtool rules load recording-rules.yaml

# 규칙 목록 확인
mimirtool rules list

# 규칙 출력 (현재 서버에 저장된 내용)
mimirtool rules print

# 로컬 파일과 서버 비교 (diff)
mimirtool rules diff recording-rules.yaml

# 규칙 삭제 (네임스페이스/그룹 지정)
mimirtool rules delete my-app "http:red"

# 규칙 린트 (문법 검사)
mimirtool rules lint recording-rules.yaml
```

---

## 5. Recording Rule 활용

```promql
# Recording Rule로 생성된 메트릭 사용
job:http_requests:rate5m{job="my-service"}

# Alerting Rule에서 Recording Rule 재사용
- alert: HighErrorRate
  expr: job:http_errors:rate5m > 0.05
  for: 5m

# Grafana 대시보드 패널 쿼리 (빠름)
job:http_request_duration_p99:rate5m{job=~"$job"}
```

---

## 6. GitOps로 규칙 관리

규칙 파일을 Git으로 버전 관리하고 CI/CD로 자동 배포:

```yaml
# .github/workflows/deploy-rules.yaml
name: Deploy Mimir Rules

on:
  push:
    branches: [main]
    paths:
      - 'rules/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install mimirtool
        run: |
          curl -Lo mimirtool https://github.com/grafana/mimir/releases/latest/download/mimirtool-linux-amd64
          chmod +x mimirtool

      - name: Lint rules
        run: ./mimirtool rules lint rules/*.yaml

      - name: Deploy rules
        env:
          MIMIR_ADDRESS: ${{ secrets.MIMIR_ADDRESS }}
          MIMIR_TENANT_ID: default
        run: ./mimirtool rules load rules/*.yaml
```

자세한 내용은 [gitops-guide.md](./gitops-guide.md) 참고.

---

## 참고 링크

- [Mimir Ruler API 문서](https://grafana.com/docs/mimir/latest/references/http-api/#ruler)
- [Recording Rules 공식 문서](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [mimirtool 공식 문서](https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/)
