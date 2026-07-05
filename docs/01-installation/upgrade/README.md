# Mimir 업그레이드 가이드

Mimir 업그레이드는 distributor, ingester, querier, query-frontend, compactor, ruler 등 여러 컴포넌트에 영향을 줍니다. ring, object storage, tenant 설정을 확인한 뒤 순차적으로 진행합니다.

## 1. 사전 점검

```bash
export NAMESPACE="monitoring"
export RELEASE="mimir"
export CHART_VERSION="5.6.0"
export VALUES_FILE="my-values.yaml"

helm status ${RELEASE} -n ${NAMESPACE}
helm history ${RELEASE} -n ${NAMESPACE}
helm get values ${RELEASE} -n ${NAMESPACE} > values-before-upgrade.yaml
kubectl get pods,svc,pvc -n ${NAMESPACE} -l app.kubernetes.io/instance=${RELEASE}
```

업그레이드 전 object storage 접근, compactor 상태, ingester ring 상태, ruler/alertmanager 사용 여부를 확인합니다.

## 2. Helm 업그레이드

```bash
helm repo update grafana
helm upgrade ${RELEASE} grafana/mimir-distributed \
  --namespace ${NAMESPACE} \
  --values ${VALUES_FILE} \
  --version ${CHART_VERSION} \
  --timeout 15m \
  --wait
```

## 3. 확인

```bash
kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/instance=${RELEASE}
kubectl port-forward svc/mimir-nginx 8080:80 -n ${NAMESPACE}
curl http://localhost:8080/ready
```

Prometheus remote_write ingestion, Grafana datasource query, ruler/alertmanager 동작을 확인합니다.

## 4. 롤백

```bash
helm history ${RELEASE} -n ${NAMESPACE}
helm rollback ${RELEASE} <REVISION> -n ${NAMESPACE} --wait
```

Mimir의 storage schema 또는 ring 동작이 변경된 major 업그레이드는 단순 rollback이 위험할 수 있습니다. 릴리즈 노트의 downgrade 가능 여부를 확인합니다.

## 5. systemd / Docker Compose

systemd 설치는 Mimir 바이너리와 gateway 설정을 백업한 뒤 서비스 단위로 교체합니다. Docker Compose 설치는 image tag를 변경하고 `docker compose pull && docker compose up -d`를 실행합니다. 단일 노드 환경도 object storage와 local data를 먼저 백업합니다.

