# Mimir Ops

Mimir를 실제로 설치하거나 실습할 때 사용하는 Helm values와 Prometheus remote_write 예시를 두는 공간입니다. 개념 설명은 `docs/`에, 적용 가능한 실행 자산은 `ops/`에 둡니다.

## 폴더 구조

| 폴더 | 내용 |
|------|------|
| `config/helm/` | 개발/HA용 Mimir Helm values |
| `config/prometheus/` | Prometheus remote_write 설정 예시 |

## 관련 문서

| 작업 | 문서 |
|------|------|
| 설치 방식 선택 | [../docs/01-installation/install.md](../docs/01-installation/install.md) |
| Helm 설치 | [../docs/01-installation/install-helm.md](../docs/01-installation/install-helm.md) |
| 스토리지 구성 | [../docs/04-storage/storage-guide.md](../docs/04-storage/storage-guide.md) |
| Remote Write 설정 | [../docs/03-ingestion/remote-write-guide.md](../docs/03-ingestion/remote-write-guide.md) |
| Grafana 조회 | [../docs/05-query/grafana-datasource-guide.md](../docs/05-query/grafana-datasource-guide.md) |
| End-to-End 실습 | [../docs/10-tutorials/e2e-practice.md](../docs/10-tutorials/e2e-practice.md) |
| 운영 진단 | [../docs/09-operations/troubleshooting-guide.md](../docs/09-operations/troubleshooting-guide.md) |

## 관리 원칙

- 재사용 가능한 Helm values와 remote_write 설정은 이 디렉터리에 둡니다.
- 문서 본문에는 핵심 스니펫만 넣고, 전체 적용 파일은 `ops/config/`를 기준으로 관리합니다.
- 설정을 바꾸면 관련 문서의 명령어와 파일 경로도 함께 확인합니다.
