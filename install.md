# Mimir 설치 (Helm)

EKS 환경에서 `mimir-distributed` Helm Chart로 Mimir를 설치합니다.

---

## 사전 조건 확인

```bash
# kubectl 연결 확인
kubectl get nodes

# Helm 버전 확인 (>= 3.10 권장)
helm version

# AWS CLI 설정 확인
aws sts get-caller-identity

# EKS OIDC Provider 활성화 여부 확인 (IRSA 필수)
aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

---

## 1. 네임스페이스 생성

```bash
kubectl create namespace monitoring
```

---

## 2. Helm 저장소 추가

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 사용 가능한 버전 확인
helm search repo grafana/mimir-distributed --versions | head -10
```

---

## 3. S3 + IRSA 설정

> **선행 작업**: [storage-guide.md](./storage-guide.md)를 먼저 완료하세요.
>
> S3 버킷 3개(blocks, ruler, alertmanager)와 IRSA 설정이 필요합니다.

---

## 4. Helm values 파일 준비

```bash
cp helm/values.yaml my-values.yaml
```

`my-values.yaml` 에서 반드시 수정할 항목:

```yaml
# 1. IRSA Role ARN
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/mimir-irsa-role

# 2. S3 버킷 이름
blocks_storage:
  s3:
    bucket_name: <YOUR_BLOCKS_BUCKET>   # 예: mycompany-mimir-blocks-prod

ruler_storage:
  s3:
    bucket_name: <YOUR_RULER_BUCKET>    # 예: mycompany-mimir-ruler-prod

alertmanager_storage:
  s3:
    bucket_name: <YOUR_AM_BUCKET>       # 예: mycompany-mimir-am-prod
```

---

## 5. Mimir 설치

```bash
# 버전 고정 권장 (운영 환경)
MIMIR_CHART_VERSION="5.5.0"

helm install mimir grafana/mimir-distributed \
  --namespace monitoring \
  --values my-values.yaml \
  --version ${MIMIR_CHART_VERSION} \
  --timeout 10m \
  --wait
```

> **팁**: `--wait` 플래그를 사용하면 모든 파드가 Ready 상태가 될 때까지 대기합니다.

---

## 6. 설치 확인

```bash
# 파드 상태 확인
kubectl get pods -n monitoring -w

# 서비스 확인
kubectl get svc -n monitoring
```

정상 설치 시 아래 컴포넌트 파드가 모두 `Running` 상태여야 합니다:

| 컴포넌트 | 기본 파드 수 | StatefulSet |
|---|---|---|
| distributor | 1 | 아니오 |
| ingester | 3 | 예 (WAL 유지) |
| querier | 2 | 아니오 |
| query-frontend | 1 | 아니오 |
| store-gateway | 1 | 예 (캐시 유지) |
| compactor | 1 | 예 |
| ruler | 1 | 아니오 |
| alertmanager | 1 | 예 |
| nginx | 1 | 아니오 |

```bash
# 빠른 상태 요약
kubectl get pods -n monitoring \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount'
```

---

## 7. 헬스 체크 (포트 포워드)

```bash
# Mimir Gateway (nginx) 포트 포워드
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring

# 전체 준비 상태 확인
curl http://localhost:8080/ready

# 컴포넌트별 상태 확인
curl http://localhost:8080/distributor/ring    # Distributor 링 상태
curl http://localhost:8080/ingester/ring       # Ingester 링 상태
curl http://localhost:8080/store-gateway/ring  # Store Gateway 링 상태
```

모든 컴포넌트가 `ready` 를 반환하면 정상입니다.

---

## 8. IRSA 연결 확인

```bash
# ServiceAccount에 IRSA 어노테이션이 붙어있는지 확인
kubectl get serviceaccount mimir -n monitoring -o yaml | grep role-arn

# Ingester 파드에서 S3 접근 테스트
kubectl exec -n monitoring -it mimir-ingester-0 -- \
  aws s3 ls s3://<YOUR_BLOCKS_BUCKET>/ --region ap-northeast-2
```

---

## 업그레이드

```bash
# 업그레이드 전 현재 values 확인
helm get values mimir -n monitoring

# 업그레이드
helm upgrade mimir grafana/mimir-distributed \
  --namespace monitoring \
  --values my-values.yaml \
  --version 5.6.0 \
  --timeout 10m

# 업그레이드 히스토리 확인
helm history mimir -n monitoring
```

> **Ingester 업그레이드 주의**: Ingester는 StatefulSet이므로 롤링 업그레이드 중
> 인메모리 데이터가 있습니다. 한 번에 1개씩 재시작되므로 quorum이 유지되는지 확인하세요.

---

## 롤백

```bash
# 이전 버전으로 롤백
helm rollback mimir 1 -n monitoring
```

---

## 삭제

```bash
helm uninstall mimir -n monitoring

# PVC 삭제 (Ingester WAL, Store Gateway 캐시 포함)
kubectl delete pvc -n monitoring -l app.kubernetes.io/instance=mimir

# 네임스페이스 삭제
kubectl delete namespace monitoring
```

> **주의**: S3 버킷의 데이터는 별도로 삭제해야 합니다.
> ```bash
> aws s3 rm s3://<YOUR_BLOCKS_BUCKET> --recursive
> ```

---

## 설치 시 자주 발생하는 문제

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| Ingester `CrashLoopBackOff` | S3 접근 실패 | IRSA ARN, 버킷 이름, 리전 확인 |
| `Error: INSTALLATION FAILED: timed out` | 리소스 부족 | 노드 용량 확인, `--timeout` 연장 |
| Store Gateway `Pending` | PVC 생성 실패 | StorageClass 확인 (`gp2` 또는 `gp3`) |
| `401 Unauthorized` on remote_write | 잘못된 설정 | `multitenancy_enabled` 값 및 헤더 확인 |
| Ingester 링에 `LEAVING` 상태 파드 | 이전 파드 정리 중 | 잠시 대기 후 재확인 |
