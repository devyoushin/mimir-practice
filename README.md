# mimir-practice

EKS 환경에서 Grafana Mimir Distributed 기반 장기 메트릭 저장소, remote write, 멀티 테넌시, S3 스토리지, 규칙/알림, 고가용성을 학습하는 문서 저장소입니다.

## 먼저 볼 문서

| 목적 | 문서 |
|------|------|
| 전체 문서 목차 보기 | [docs/README.md](docs/README.md) |
| Mimir 설치하기 | [docs/install/install.md](docs/install/install.md) |
| 아키텍처 이해하기 | [docs/architecture/architecture-guide.md](docs/architecture/architecture-guide.md) |
| S3/IRSA 스토리지 구성하기 | [docs/storage/storage-guide.md](docs/storage/storage-guide.md) |
| 멀티 테넌시 이해하기 | [docs/tenancy/multi-tenancy-guide.md](docs/tenancy/multi-tenancy-guide.md) |
| Prometheus remote write 연결하기 | [docs/ingestion/remote-write-guide.md](docs/ingestion/remote-write-guide.md) |
| Grafana에서 조회하기 | [docs/query/grafana-datasource-guide.md](docs/query/grafana-datasource-guide.md) |
| End-to-End 실습하기 | [docs/tutorials/e2e-practice.md](docs/tutorials/e2e-practice.md) |
| 운영 YAML 확인하기 | [ops/README.md](ops/README.md) |

## 추천 학습 순서

1. [Mimir 설치](docs/install/install.md)
2. [아키텍처](docs/architecture/architecture-guide.md)
3. [스토리지](docs/storage/storage-guide.md)
4. [멀티 테넌시](docs/tenancy/multi-tenancy-guide.md)
5. [Remote Write](docs/ingestion/remote-write-guide.md)
6. [Grafana 데이터소스](docs/query/grafana-datasource-guide.md)
7. [Recording Rules](docs/rules-alerting/recording-rules-guide.md), [Alerting](docs/rules-alerting/alerting-guide.md)
8. [Compactor](docs/operations/compactor-guide.md), [HA](docs/operations/ha-guide.md), [Monitoring](docs/operations/monitoring-guide.md)
9. [GitOps](docs/delivery/gitops-guide.md), [End-to-End 실습](docs/tutorials/e2e-practice.md)
10. [트러블슈팅](docs/operations/troubleshooting-guide.md)

## 디렉터리 구조

```text
mimir-practice/
├── README.md
├── CLAUDE.md          # AI 작업 지침
├── docs/
│   ├── README.md     # 문서 전체 목차
│   ├── install/      # 설치와 업그레이드
│   ├── architecture/ # Mimir 컴포넌트와 read/write path
│   ├── storage/      # S3, IRSA, 블록 저장소
│   ├── tenancy/      # 멀티 테넌시와 tenant limits
│   ├── ingestion/    # Prometheus remote_write
│   ├── query/        # Grafana 데이터소스와 조회
│   ├── rules-alerting/ # Recording Rules, Alerting
│   ├── operations/   # Compactor, HA, monitoring, troubleshooting
│   ├── delivery/     # GitOps 기반 규칙 관리
│   ├── tutorials/    # End-to-End 실습
│   ├── agents/       # AI 역할별 작업 지침
│   ├── rules/        # 문서/운영 규칙
│   └── templates/    # 서비스 문서, 런북, 장애 보고서 템플릿
└── ops/
    ├── README.md
    └── config/       # Mimir Helm values와 Prometheus remote_write 예시
```

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Chart | `grafana/mimir-distributed` |
| Namespace | `monitoring` |
| Storage | S3 + IRSA |
| Region | `ap-northeast-2` |
