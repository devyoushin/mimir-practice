# Grafana Mimir on EKS — 실습 저장소

EKS 환경에서 Grafana Mimir를 처음부터 운영 수준까지 학습하는 실습 저장소입니다.

---

## 환경 정보

| 항목 | 값 |
|---|---|
| 플랫폼 | AWS EKS |
| Mimir Helm Chart | `grafana/mimir-distributed` (권장 버전: 5.5.x) |
| 네임스페이스 | `monitoring` |
| 스토리지 | S3 + IRSA 인증 |
| 리전 | `ap-northeast-2` |

---

## 사전 요구사항

```bash
# 필요 도구 확인
kubectl version --client      # >= 1.25
helm version                  # >= 3.10
aws --version                 # AWS CLI v2
eksctl version                # IRSA 설정용

# EKS 클러스터 접속 확인
kubectl get nodes
```

---

## 빠른 시작 (Quick Start)

```bash
# 1. S3 버킷 생성 (storage-guide.md 참고)
# 2. IRSA 설정 (storage-guide.md 참고)

# 3. Helm values 수정
cp helm/values.yaml my-values.yaml
# my-values.yaml에서 버킷 이름, 리전, IRSA ARN 수정

# 4. Mimir 설치
kubectl create namespace monitoring
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
helm install mimir grafana/mimir-distributed \
  --namespace monitoring \
  --values my-values.yaml \
  --version 5.5.0

# 5. 헬스 체크
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring &
curl http://localhost:8080/ready
```

---

## 학습 경로

### 1단계: 설치
- [Helm으로 Mimir 설치](./install.md)

### 2단계: 핵심 개념
- [아키텍처 개요](./architecture-guide.md)
- [스토리지 구성 (S3 + IRSA)](./storage-guide.md)
- [멀티 테넌시](./multi-tenancy-guide.md)

### 3단계: 데이터 흐름
- [Write Path — Prometheus Remote Write](./remote-write-guide.md)
- [Read Path — Grafana 연동](./grafana-datasource-guide.md)

### 4단계: 규칙 & 알림
- [Recording Rules](./recording-rules-guide.md)
- [Alerting Rules & Alertmanager](./alerting-guide.md)

### 5단계: 운영 심화
- [Compactor — 블록 최적화](./compactor-guide.md)
- [고가용성 (Zone-aware Replication)](./ha-guide.md)
- [메타 모니터링 (Mimir로 Mimir 관찰)](./monitoring-guide.md)
- [GitOps로 Rules 관리](./gitops-guide.md)

### 6단계: 문제 해결
- [트러블슈팅 가이드](./troubleshooting-guide.md)

### 실습
- [End-to-End 실습 (Prometheus → Mimir → Grafana)](./e2e-practice.md)

---

## 저장소 구조

```
mimir-practice/
├── README.md
├── install.md                   # Helm 설치 가이드
├── architecture-guide.md        # 아키텍처 설명
├── storage-guide.md             # S3 + IRSA 구성
├── multi-tenancy-guide.md       # 멀티 테넌시
├── remote-write-guide.md        # Prometheus → Mimir 쓰기
├── grafana-datasource-guide.md  # Grafana 연동
├── recording-rules-guide.md     # Recording Rules
├── alerting-guide.md            # Alerting + Alertmanager
├── compactor-guide.md           # 블록 압축
├── ha-guide.md                  # 고가용성
├── monitoring-guide.md          # 메타 모니터링
├── gitops-guide.md              # GitOps 규칙 관리
├── troubleshooting-guide.md     # 트러블슈팅
├── e2e-practice.md              # End-to-End 실습
├── helm/
│   ├── values.yaml              # 개발/테스트용 Helm values
│   └── values-ha.yaml           # 운영 HA Helm values
└── prometheus/
    └── remote-write.yaml        # Prometheus remote_write 설정 예시
```

---

## 아키텍처 요약

```
Prometheus ──remote_write──▶ Distributor ──▶ Ingester ──▶ S3 (TSDB)
(X-Scope-OrgID: tenant)          │                            │
                          consistent hash              Store Gateway
                                                            │
Grafana ◀──── Query Frontend ◀──── Querier ◀───────────────┘
(X-Scope-OrgID: tenant)                 └─── Ingester (최근 데이터)

                Ruler ──▶ Recording/Alerting Rules ──▶ Alertmanager
```

| 컴포넌트 | 역할 |
|---|---|
| **Distributor** | remote_write 수신, 유효성 검증, Ingester에 분산 |
| **Ingester** | 인메모리 버퍼, WAL, 주기적 S3 플러시 |
| **Querier** | Ingester + Store Gateway 데이터 합산 쿼리 |
| **Query Frontend** | 쿼리 캐싱, 샤딩, 병렬화 |
| **Store Gateway** | S3 블록 인덱스 캐싱, 범위 쿼리 최적화 |
| **Compactor** | S3 블록 병합, 만료 데이터 삭제 |
| **Ruler** | Recording/Alerting Rules 실행 |
| **Alertmanager** | 알림 라우팅, 중복 제거, Slack/PD 발송 |

---

## 참고 링크

- [Grafana Mimir 공식 문서](https://grafana.com/docs/mimir/latest/)
- [mimir-distributed Helm Chart](https://github.com/grafana/mimir/tree/main/operations/helm/charts/mimir-distributed)
- [Helm Chart Changelog](https://github.com/grafana/mimir/blob/main/operations/helm/charts/mimir-distributed/CHANGELOG.md)
- [mimirtool CLI](https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/)
