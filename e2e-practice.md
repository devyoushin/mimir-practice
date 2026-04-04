# End-to-End 실습: Prometheus → Mimir → Grafana

설치부터 알림 발송까지 전체 흐름을 실습합니다.

---

## 사전 준비 체크리스트

- [ ] EKS 클러스터 접속 확인: `kubectl get nodes`
- [ ] AWS CLI 설정 확인: `aws sts get-caller-identity`
- [ ] Helm 설치 확인: `helm version`
- [ ] S3 버킷 3개 생성 완료 ([storage-guide.md](./storage-guide.md) 참고)
- [ ] IRSA 설정 완료 ([storage-guide.md](./storage-guide.md) 참고)

---

## Step 1: Mimir 설치

```bash
# 네임스페이스 생성
kubectl create namespace monitoring

# Helm repo 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# values.yaml 복사 후 수정
cp helm/values.yaml my-values.yaml

# 필수 수정 항목:
# 1. serviceAccount.annotations.eks.amazonaws.com/role-arn
# 2. blocks_storage.s3.bucket_name
# 3. ruler_storage.s3.bucket_name
# 4. alertmanager_storage.s3.bucket_name
vi my-values.yaml

# 설치
helm install mimir grafana/mimir-distributed \
  --namespace monitoring \
  --values my-values.yaml \
  --version 5.5.0 \
  --timeout 10m \
  --wait

# 설치 확인
kubectl get pods -n monitoring
```

**예상 결과**: 모든 파드가 `Running` 상태

---

## Step 2: 헬스 체크

```bash
# Mimir Gateway 포트 포워드 (별도 터미널에서 실행)
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring &

# 전체 준비 상태 확인
curl -s http://localhost:8080/ready
# 예상 출력: ready

# 각 컴포넌트 링 상태 확인
curl -s http://localhost:8080/distributor/ring | jq '.shards | length'
# 예상 출력: 1 (기본 설정)

curl -s http://localhost:8080/ingester/ring | jq '.shards[] | {id: .id, state: .state}'
# 예상 출력: 3개 Ingester 모두 ACTIVE
```

---

## Step 3: Prometheus 배포 + Remote Write 설정

```bash
# Prometheus 설치 (kube-prometheus-stack 사용 — Grafana 포함)
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.remoteWrite[0].url="http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push" \
  --set "prometheus.prometheusSpec.remoteWrite[0].headers.X-Scope-OrgID=default" \
  --set grafana.enabled=false \   # Grafana는 별도로 설치할 경우
  --wait

# 또는 단순 Prometheus만
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.remoteWrite[0].url="http://mimir-nginx.monitoring.svc.cluster.local/api/v1/push" \
  --set "server.remoteWrite[0].headers.X-Scope-OrgID=default"
```

---

## Step 4: 데이터 수신 확인

```bash
# 2~3분 대기 (scrape 및 remote_write 안정화)
sleep 120

# Mimir에서 메트릭 조회
curl -s "http://localhost:8080/prometheus/api/v1/query?query=up" \
  -H "X-Scope-OrgID: default" | jq '.data.result | length'
# 예상 출력: 1 이상 (수집 중인 타겟 수)

# 수신된 샘플 수 확인
curl -s "http://localhost:8080/prometheus/api/v1/query?query=sum(cortex_ingester_memory_series)" \
  -H "X-Scope-OrgID: default" | jq '.data.result[0].value[1]'
# 예상 출력: 1000 이상 (활성 시계열 수)

# remote_write 에러율 확인 (0이어야 함)
curl -s "http://localhost:8080/prometheus/api/v1/query?query=rate(cortex_distributor_ingestion_failures_total[5m])" \
  -H "X-Scope-OrgID: default" | jq '.data.result'
```

---

## Step 5: Grafana 배포 + Mimir 데이터소스 연동

```bash
# Grafana 설치
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set adminPassword=admin123 \
  --set "datasources.datasources\\.yaml.apiVersion=1" \
  --wait

# 또는 datasource 파일로 설정
cat > grafana-values.yaml << 'EOF'
persistence:
  enabled: true

adminPassword: admin123

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Mimir
        type: prometheus
        url: http://mimir-nginx.monitoring.svc.cluster.local/prometheus
        access: proxy
        httpMethod: POST
        jsonData:
          prometheusType: Mimir
          httpHeaderName1: X-Scope-OrgID
        secureJsonData:
          httpHeaderValue1: default
        isDefault: true
EOF

helm upgrade grafana grafana/grafana \
  --namespace monitoring \
  --values grafana-values.yaml

# Grafana 포트 포워드 (별도 터미널)
kubectl port-forward svc/grafana 3000:80 -n monitoring &
```

브라우저에서 `http://localhost:3000` 접속 (admin / admin123)

**데이터소스 테스트:**

1. Configuration → Data Sources → Mimir
2. "Save & Test" 클릭
3. "Data source is working" 확인

---

## Step 6: Recording Rules 등록

```bash
# Recording Rules 업로드
curl -X POST \
  http://localhost:8080/prometheus/config/v1/rules/practice \
  -H "X-Scope-OrgID: default" \
  -H "Content-Type: application/yaml" \
  --data-binary @- << 'EOF'
groups:
  - name: practice.rules
    interval: 1m
    rules:
      - record: job:up:sum
        expr: sum by (job) (up)

      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: cluster:up:ratio
        expr: sum(up) / count(up)
EOF

# 등록 확인 (약 1분 후 평가됨)
curl -s http://localhost:8080/prometheus/api/v1/rules \
  -H "X-Scope-OrgID: default" | jq '.data.groups[].rules[].name'

# 1분 후 결과 조회
sleep 60
curl -s "http://localhost:8080/prometheus/api/v1/query?query=job:up:sum" \
  -H "X-Scope-OrgID: default" | jq '.data.result'
```

---

## Step 7: Alerting Rules + Alertmanager 설정

```bash
# Alerting Rules 업로드
curl -X POST \
  http://localhost:8080/prometheus/config/v1/rules/practice-alerts \
  -H "X-Scope-OrgID: default" \
  -H "Content-Type: application/yaml" \
  --data-binary @- << 'EOF'
groups:
  - name: practice.alerts
    rules:
      - alert: AlwaysFiringTest
        expr: vector(1) == 1
        for: 0m
        labels:
          severity: info
        annotations:
          summary: "테스트 알림 - 항상 발동"
          description: "이 알림은 Alertmanager 연동 테스트용입니다."
EOF

# 30초 후 알림 상태 확인
sleep 30
curl -s http://localhost:8080/prometheus/api/v1/alerts \
  -H "X-Scope-OrgID: default" | \
  jq '.data.alerts[] | select(.labels.alertname == "AlwaysFiringTest")'
```

---

## Step 8: Mimir 모니터링 대시보드 Import

Grafana에서 아래 대시보드를 Import하세요:

| 대시보드 | ID |
|---|---|
| Mimir Overview | 15869 |
| Mimir Writes | 15872 |
| Mimir Reads | 15870 |

```bash
GRAFANA_URL="http://localhost:3000"

for ID in 15869 15870 15872; do
  curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
    -u admin:admin123 \
    -H "Content-Type: application/json" \
    -d "{
      \"gnetId\": ${ID},
      \"overwrite\": true,
      \"inputs\": [{
        \"name\": \"DS_PROMETHEUS\",
        \"type\": \"datasource\",
        \"pluginId\": \"prometheus\",
        \"value\": \"Mimir\"
      }]
    }"
  echo "Imported dashboard ${ID}"
done
```

---

## 검증 체크리스트

```bash
# 자동화 검증 스크립트
echo "=== Mimir E2E 검증 ==="

# 1. Mimir 파드 상태
echo -n "[1] Mimir 파드 모두 Running: "
RUNNING=$(kubectl get pods -n monitoring -l app.kubernetes.io/instance=mimir \
  --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
TOTAL=$(kubectl get pods -n monitoring -l app.kubernetes.io/instance=mimir \
  --no-headers 2>/dev/null | wc -l)
echo "${RUNNING}/${TOTAL}"

# 2. Mimir ready 상태
echo -n "[2] Mimir ready: "
curl -s http://localhost:8080/ready

# 3. 메트릭 수신 여부
echo ""
echo -n "[3] 수신 중인 시계열 수: "
curl -s "http://localhost:8080/prometheus/api/v1/query?query=sum(cortex_ingester_memory_series)" \
  -H "X-Scope-OrgID: default" | jq -r '.data.result[0].value[1]'

# 4. Remote write 에러 없음
echo -n "[4] Ingestion 실패 여부: "
FAILURES=$(curl -s "http://localhost:8080/prometheus/api/v1/query?query=rate(cortex_distributor_ingestion_failures_total[5m])" \
  -H "X-Scope-OrgID: default" | jq -r '.data.result | length')
if [ "$FAILURES" -eq 0 ]; then echo "OK (에러 없음)"; else echo "WARNING: ${FAILURES}개 실패"; fi

# 5. Recording Rules 평가 중
echo -n "[5] Recording Rules 평가 중: "
curl -s http://localhost:8080/prometheus/api/v1/rules \
  -H "X-Scope-OrgID: default" | jq -r '.data.groups[0].rules | length'

# 6. Grafana 접속 가능
echo -n "[6] Grafana 응답: "
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/health

echo ""
echo "=== 검증 완료 ==="
```

---

## 실습 종료 후 정리

```bash
# Helm 릴리즈 삭제
helm uninstall mimir -n monitoring
helm uninstall kube-prometheus-stack -n monitoring
helm uninstall grafana -n monitoring

# PVC 삭제 (Ingester WAL 등)
kubectl delete pvc -n monitoring --all

# 네임스페이스 삭제
kubectl delete namespace monitoring

# S3 버킷 데이터 삭제 (비용 발생 주의)
aws s3 rm s3://mycompany-mimir-blocks --recursive
aws s3 rm s3://mycompany-mimir-ruler --recursive
aws s3 rm s3://mycompany-mimir-alertmanager --recursive

# 포트 포워드 프로세스 종료
pkill -f "kubectl port-forward"
```

---

## 자주 막히는 지점

| 단계 | 증상 | 해결 |
|---|---|---|
| Step 1 | Ingester `Init:0/1` | IRSA ARN 또는 S3 버킷 이름 확인 |
| Step 1 | Store Gateway `Pending` | StorageClass 존재 여부 확인 |
| Step 4 | 결과가 0개 | `X-Scope-OrgID` 헤더 값이 remote_write와 동일한지 확인 |
| Step 5 | "Data source not responding" | mimir-nginx 서비스 이름/네임스페이스 확인 |
| Step 6 | Recording Rule 결과 없음 | 1분 대기 후 재시도 |

더 자세한 문제 해결은 [troubleshooting-guide.md](./troubleshooting-guide.md) 참고.
