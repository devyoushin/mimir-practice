# Mimir 설치 (Helm)

EKS 환경에서 `mimir-distributed` Helm Chart로 Mimir를 설치한다.

## 사전 준비

```bash
kubectl get nodes
helm version
aws sts get-caller-identity
kubectl create namespace monitoring
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## 선행 조건

- [../04-storage/storage-guide.md](../04-storage/storage-guide.md) 먼저 완료
- blocks / ruler / alertmanager S3 버킷과 IRSA 필요

## 설치

```bash
MIMIR_CHART_VERSION="5.5.0"
helm install mimir grafana/mimir-distributed \
  --namespace monitoring \
  --values my-values.yaml \
  --version ${MIMIR_CHART_VERSION} \
  --timeout 10m \
  --wait
```

## 확인

```bash
kubectl get pods -n monitoring -w
kubectl get svc -n monitoring
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring
curl http://localhost:8080/ready
```

## 업그레이드 / 롤백 / 삭제

```bash
helm upgrade mimir grafana/mimir-distributed -n monitoring --values my-values.yaml --version 5.6.0 --timeout 10m
helm rollback mimir 1 -n monitoring
helm uninstall mimir -n monitoring
kubectl delete pvc -n monitoring -l app.kubernetes.io/instance=mimir
```
