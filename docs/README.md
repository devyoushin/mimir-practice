# Mimir Docs

Mimir를 처음 보는 사람이 설치부터 멀티 테넌시와 운영까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install/install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `install/upgrade/` | Mimir 업그레이드 |
| 3 | `architecture-guide.md` | 컴포넌트 구조와 read/write path |
| 4 | `storage-guide.md` | S3, IRSA, 블록 저장소 |
| 5 | `multi-tenancy-guide.md` | tenant 격리와 `X-Scope-OrgID` |
| 6 | `remote-write-guide.md`, `grafana-datasource-guide.md` | 데이터 흐름과 시각화 연동 |
| 7 | `recording-rules-guide.md`, `alerting-guide.md` | 규칙과 알림 |
| 8 | `compactor-guide.md`, `ha-guide.md`, `monitoring-guide.md` | 운영과 고가용성 |
| 9 | `gitops-guide.md`, `e2e-practice.md` | GitOps와 실습 |
| 10 | `rules/README.md` | 문서와 운영 규칙 |
| 11 | `agents/README.md` | AI 작업 지침 |
| 12 | `templates/README.md` | 문서 템플릿 |
| 13 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/기초 | `install/install.md`, `install/upgrade/`, `architecture-guide.md`, `storage-guide.md` |
| 데이터 흐름 | `multi-tenancy-guide.md`, `remote-write-guide.md`, `grafana-datasource-guide.md` |
| 규칙/알림 | `recording-rules-guide.md`, `alerting-guide.md` |
| 운영 | `compactor-guide.md`, `ha-guide.md`, `monitoring-guide.md`, `troubleshooting-guide.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install/install.md`
2. `install/upgrade/`
3. `architecture-guide.md`
4. `storage-guide.md`
5. `multi-tenancy-guide.md`
6. `remote-write-guide.md`
7. `grafana-datasource-guide.md`
8. `recording-rules-guide.md`
9. `alerting-guide.md`
10. `compactor-guide.md`
11. `ha-guide.md`
12. `rules/README.md`
13. `agents/README.md`
14. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Helm values와 remote_write 예시
