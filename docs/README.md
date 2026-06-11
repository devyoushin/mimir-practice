# Mimir Docs

Mimir를 처음 보는 사람이 설치, 아키텍처, 스토리지, 멀티 테넌시, remote write, 조회, 규칙/알림, 운영까지 순서대로 따라갈 수 있도록 정리한 문서 디렉터리입니다.

## 빠른 길잡이

| 지금 하고 싶은 일 | 열 문서 |
|------|------|
| 설치 방식을 고르기 | [install/install.md](install/install.md) |
| 컴포넌트와 read/write path 이해하기 | [architecture/architecture-guide.md](architecture/architecture-guide.md) |
| S3와 IRSA 스토리지 구성하기 | [storage/storage-guide.md](storage/storage-guide.md) |
| tenant 격리와 `X-Scope-OrgID` 이해하기 | [tenancy/multi-tenancy-guide.md](tenancy/multi-tenancy-guide.md) |
| Prometheus remote_write 연결하기 | [ingestion/remote-write-guide.md](ingestion/remote-write-guide.md) |
| Grafana 데이터소스 설정하기 | [query/grafana-datasource-guide.md](query/grafana-datasource-guide.md) |
| Recording Rule과 Alerting Rule 만들기 | [rules-alerting/recording-rules-guide.md](rules-alerting/recording-rules-guide.md), [rules-alerting/alerting-guide.md](rules-alerting/alerting-guide.md) |
| Compactor, HA, 모니터링 운영하기 | [operations/compactor-guide.md](operations/compactor-guide.md), [operations/ha-guide.md](operations/ha-guide.md), [operations/monitoring-guide.md](operations/monitoring-guide.md) |
| GitOps로 규칙 관리하기 | [delivery/gitops-guide.md](delivery/gitops-guide.md) |
| End-to-End로 검증하기 | [tutorials/e2e-practice.md](tutorials/e2e-practice.md) |
| 장애를 진단하기 | [operations/troubleshooting-guide.md](operations/troubleshooting-guide.md) |

## 추천 읽기 순서

| 순서 | 문서 | 핵심 내용 |
|------|------|------|
| 1 | [install/install.md](install/install.md) | Helm, systemd, Docker Compose 설치 방식 |
| 2 | [architecture/architecture-guide.md](architecture/architecture-guide.md) | 컴포넌트 구조와 read/write path |
| 3 | [storage/storage-guide.md](storage/storage-guide.md) | S3, IRSA, 블록 저장소 |
| 4 | [tenancy/multi-tenancy-guide.md](tenancy/multi-tenancy-guide.md) | tenant 격리와 tenant별 limits |
| 5 | [ingestion/remote-write-guide.md](ingestion/remote-write-guide.md) | Prometheus remote_write 전송 |
| 6 | [query/grafana-datasource-guide.md](query/grafana-datasource-guide.md) | Grafana 데이터소스와 조회 |
| 7 | [rules-alerting/recording-rules-guide.md](rules-alerting/recording-rules-guide.md) | Recording Rules |
| 8 | [rules-alerting/alerting-guide.md](rules-alerting/alerting-guide.md) | Ruler와 Alertmanager |
| 9 | [operations/compactor-guide.md](operations/compactor-guide.md) | Compactor 운영 |
| 10 | [operations/ha-guide.md](operations/ha-guide.md) | 고가용성 구성 |
| 11 | [operations/monitoring-guide.md](operations/monitoring-guide.md) | Mimir 메타 모니터링 |
| 12 | [delivery/gitops-guide.md](delivery/gitops-guide.md) | GitOps 기반 규칙 관리 |
| 13 | [tutorials/e2e-practice.md](tutorials/e2e-practice.md) | End-to-End 실습 |
| 14 | [operations/troubleshooting-guide.md](operations/troubleshooting-guide.md) | 증상별 트러블슈팅 |

## 전체 문서 목록

| 구분 | 문서 |
|------|------|
| 설치 | [install/install.md](install/install.md), [install/install-helm.md](install/install-helm.md), [install/install-systemd.md](install/install-systemd.md), [install/install-docker-compose.md](install/install-docker-compose.md), [install/upgrade/README.md](install/upgrade/README.md) |
| 아키텍처 | [architecture/architecture-guide.md](architecture/architecture-guide.md) |
| 스토리지 | [storage/storage-guide.md](storage/storage-guide.md) |
| 멀티 테넌시 | [tenancy/multi-tenancy-guide.md](tenancy/multi-tenancy-guide.md) |
| 데이터 수집 | [ingestion/remote-write-guide.md](ingestion/remote-write-guide.md) |
| 조회/시각화 | [query/grafana-datasource-guide.md](query/grafana-datasource-guide.md) |
| 규칙/알림 | [rules-alerting/recording-rules-guide.md](rules-alerting/recording-rules-guide.md), [rules-alerting/alerting-guide.md](rules-alerting/alerting-guide.md) |
| 운영 | [operations/compactor-guide.md](operations/compactor-guide.md), [operations/ha-guide.md](operations/ha-guide.md), [operations/monitoring-guide.md](operations/monitoring-guide.md), [operations/troubleshooting-guide.md](operations/troubleshooting-guide.md) |
| 배포 관리 | [delivery/gitops-guide.md](delivery/gitops-guide.md) |
| 실습 | [tutorials/e2e-practice.md](tutorials/e2e-practice.md) |
| 문서 운영 | [rules/README.md](rules/README.md), [templates/README.md](templates/README.md), [agents/README.md](agents/README.md) |
| 실행 자산 | [../ops/README.md](../ops/README.md) |

## 폴더 역할

| 폴더 | 역할 |
|------|------|
| [install/](install/README.md) | 설치 방식별 절차와 업그레이드 |
| [architecture/](architecture/README.md) | Mimir 컴포넌트와 read/write path |
| [storage/](storage/README.md) | S3, IRSA, 블록 저장소 |
| [tenancy/](tenancy/README.md) | 멀티 테넌시와 tenant별 limits |
| [ingestion/](ingestion/README.md) | Prometheus remote_write 수집 |
| [query/](query/README.md) | Grafana 데이터소스와 조회 |
| [rules-alerting/](rules-alerting/README.md) | Recording Rules, Alerting Rules, Alertmanager |
| [operations/](operations/README.md) | Compactor, HA, monitoring, troubleshooting |
| [delivery/](delivery/README.md) | GitOps 기반 규칙 관리 |
| [tutorials/](tutorials/README.md) | End-to-End 실습 |
| [rules/](rules/README.md) | 문서 작성, Mimir 컨벤션, 보안, 모니터링 규칙 |
| [templates/](templates/README.md) | 서비스 문서, 런북, 장애 보고서 템플릿 |
| [agents/](agents/README.md) | AI가 문서를 작성하거나 진단할 때 참고할 역할별 지침 |

## 관리 원칙

- 설치는 `install/`, 구조 설명은 `architecture/`, 저장소 구성은 `storage/`, tenant 설계는 `tenancy/`에 둡니다.
- 데이터 수집은 `ingestion/`, 조회/시각화는 `query/`, 규칙과 알림은 `rules-alerting/`에 둡니다.
- 운영 절차는 `operations/`, GitOps와 배포 관리는 `delivery/`, 직접 따라 하는 검증은 `tutorials/`에 둡니다.
- 실제 적용 가능한 Helm values와 remote_write 설정은 `ops/`에 둡니다.
