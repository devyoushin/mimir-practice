# 스토리지 구성 (S3 + IRSA)

Mimir는 장기 메트릭 저장소로 S3 호환 오브젝트 스토리지를 사용합니다.
EKS 환경에서는 **IRSA (IAM Roles for Service Accounts)** 로 인증하여 액세스 키 없이 S3에 접근합니다.

---

## 버킷 설계 원칙

Mimir는 기능별로 **버킷을 3개로 분리**하는 것을 권장합니다:

| 버킷 용도 | 설명 | 크기 예측 |
|---|---|---|
| `blocks` | TSDB 블록 (메트릭 데이터 본체) | 가장 큼 (GB ~ TB) |
| `ruler` | Recording/Alerting Rules 파일 | 매우 작음 (수 MB) |
| `alertmanager` | Alertmanager 설정 파일 | 매우 작음 (수 KB) |

---

## 1. S3 버킷 생성

```bash
REGION="ap-northeast-2"
PREFIX="mycompany-mimir"   # 회사/팀 이름으로 수정

# blocks 버킷
aws s3api create-bucket \
  --bucket ${PREFIX}-blocks \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION}

# ruler 버킷
aws s3api create-bucket \
  --bucket ${PREFIX}-ruler \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION}

# alertmanager 버킷
aws s3api create-bucket \
  --bucket ${PREFIX}-alertmanager \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION}
```

### 버킷 보안 설정 (권장)

```bash
# 퍼블릭 액세스 차단
for BUCKET in ${PREFIX}-blocks ${PREFIX}-ruler ${PREFIX}-alertmanager; do
  aws s3api put-public-access-block \
    --bucket ${BUCKET} \
    --public-access-block-configuration \
      "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
done

# 서버 사이드 암호화 활성화 (SSE-S3)
for BUCKET in ${PREFIX}-blocks ${PREFIX}-ruler ${PREFIX}-alertmanager; do
  aws s3api put-bucket-encryption \
    --bucket ${BUCKET} \
    --server-side-encryption-configuration '{
      "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
    }'
done
```

### S3 Lifecycle 정책 (비용 최적화)

retention 기간 이후 자동 삭제를 S3 수준에서도 설정하면 안전합니다:

```bash
# 90일 보존 예시 (Mimir retention과 동일하게)
aws s3api put-bucket-lifecycle-configuration \
  --bucket ${PREFIX}-blocks \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "delete-old-blocks",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Expiration": {"Days": 95}
    }]
  }'
```

---

## 2. IAM 정책 생성

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MimirS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::mycompany-mimir-blocks",
        "arn:aws:s3:::mycompany-mimir-blocks/*",
        "arn:aws:s3:::mycompany-mimir-ruler",
        "arn:aws:s3:::mycompany-mimir-ruler/*",
        "arn:aws:s3:::mycompany-mimir-alertmanager",
        "arn:aws:s3:::mycompany-mimir-alertmanager/*"
      ]
    }
  ]
}
```

```bash
# 정책 파일 저장 후 생성
cat > mimir-s3-policy.json << 'EOF'
{위 JSON 내용}
EOF

aws iam create-policy \
  --policy-name MimirS3Policy \
  --policy-document file://mimir-s3-policy.json

# ARN 확인
aws iam list-policies --query 'Policies[?PolicyName==`MimirS3Policy`].Arn' --output text
```

---

## 3. IRSA 설정

```bash
CLUSTER_NAME="my-eks-cluster"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# OIDC Provider 확인 (없으면 아래 명령으로 생성)
aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query "cluster.identity.oidc.issuer" \
  --output text

# OIDC Provider 생성 (아직 없는 경우)
eksctl utils associate-iam-oidc-provider \
  --cluster ${CLUSTER_NAME} \
  --approve

# IRSA ServiceAccount 생성
eksctl create iamserviceaccount \
  --cluster ${CLUSTER_NAME} \
  --namespace monitoring \
  --name mimir \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/MimirS3Policy \
  --approve \
  --override-existing-serviceaccounts
```

### IRSA 검증

```bash
# ServiceAccount에 IRSA 어노테이션 확인
kubectl get serviceaccount mimir -n monitoring -o yaml

# 예상 출력:
# metadata:
#   annotations:
#     eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/eksctl-...-mimir
```

---

## 4. Helm values에 S3 설정 반영

```yaml
# ../ops/config/helm/values.yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/mimir-irsa-role

mimir:
  structuredConfig:
    common:
      storage:
        backend: s3
        s3:
          region: ap-northeast-2
          # IRSA 사용 시 access_key / secret_key 입력 불필요
          # (환경변수로 자동 주입됨)

    blocks_storage:
      s3:
        bucket_name: mycompany-mimir-blocks
      tsdb:
        retention_period: 13h   # Ingester flush 주기(2h)보다 충분히 길게

    ruler_storage:
      s3:
        bucket_name: mycompany-mimir-ruler

    alertmanager_storage:
      s3:
        bucket_name: mycompany-mimir-alertmanager
```

---

## 5. 스토리지 상태 확인

### Ingester가 S3에 블록을 플러시했는지 확인

```bash
# 버킷 내 블록 목록 (테넌트별 디렉토리 구조)
aws s3 ls s3://mycompany-mimir-blocks/ --recursive | head -20

# 예상 출력:
# default/01ABCDEFGHJK.../chunks/000001
# default/01ABCDEFGHJK.../index
# default/01ABCDEFGHJK.../meta.json
# default/01ABCDEFGHJK.../tombstones
```

### S3 블록 메타데이터 확인

```bash
# 블록 ID (ULID 형식) 목록 확인
aws s3 ls s3://mycompany-mimir-blocks/default/

# 특정 블록 meta.json 확인 (블록 시간 범위, 시계열 수 등)
aws s3 cp s3://mycompany-mimir-blocks/default/<BLOCK_ULID>/meta.json - | jq
```

---

## 데이터 보존 (Retention) 설정

```yaml
mimir:
  structuredConfig:
    compactor:
      retention_period: 90d   # 전역 기본값 (0 = 무기한)

    limits:
      compactor_blocks_retention_period: 90d
```

### 테넌트별 Retention 오버라이드

```yaml
# runtime-config.yaml
overrides:
  team-a:
    compactor_blocks_retention_period: 180d   # 장기 보존
  team-b:
    compactor_blocks_retention_period: 14d    # 단기 보존
```

> **비용 계산 기준**: S3 Standard 기준 약 $0.023/GB/월
> 1,000개 시계열 × 15초 간격 × 90일 ≈ 약 5GB
> 10만 시계열 × 15초 간격 × 90일 ≈ 약 500GB

---

## VPC Endpoint 설정 (보안/비용 최적화)

EKS → S3 트래픽이 인터넷을 거치지 않도록 VPC Endpoint를 설정하세요:

```bash
# S3 Gateway Endpoint 생성
aws ec2 create-vpc-endpoint \
  --vpc-id <YOUR_VPC_ID> \
  --service-name com.amazonaws.ap-northeast-2.s3 \
  --route-table-ids <ROUTE_TABLE_IDS> \
  --vpc-endpoint-type Gateway
```

> Gateway Endpoint는 무료이며 데이터 전송 비용도 절감됩니다.

---

## 트러블슈팅

```bash
# Ingester 파드에서 S3 접근 오류 확인
kubectl logs -n monitoring -l app.kubernetes.io/component=ingester \
  | grep -i "s3\|access denied\|NoSuchBucket"

# 임시 권한 테스트 (파드 내에서)
kubectl exec -n monitoring mimir-ingester-0 -- \
  env | grep AWS

# 예상: AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE 환경변수 존재
```

| 오류 메시지 | 원인 | 해결 |
|---|---|---|
| `AccessDenied` | IAM 정책 미적용 또는 IRSA 설정 오류 | IAM 정책과 IRSA ARN 재확인 |
| `NoSuchBucket` | 버킷 이름 오타 또는 리전 불일치 | values.yaml의 bucket_name, region 확인 |
| `SlowDown` | S3 요청 과다 (초기 cold start) | 잠시 대기 후 재확인 |
