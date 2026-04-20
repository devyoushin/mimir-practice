# mimir-practice — 프로젝트 가이드

## 프로젝트 설정
- 환경: EKS
- Mimir 버전: 2.x (mimir-distributed Helm chart)
- 네임스페이스: monitoring
- 스토리지: S3 (IRSA 인증)
- 앱 이름: mimir

---

## 디렉토리 구조

```
mimir-practice/
├── CLAUDE.md                  # 이 파일 (자동 로드)
├── .claude/
│   ├── settings.json
│   └── commands/              # /new-doc, /new-runbook, /review-doc, /add-troubleshooting, /search-kb
├── agents/                    # doc-writer, capacity-planner, multi-tenancy-advisor, troubleshooter
├── templates/                 # service-doc, runbook, incident-report
├── rules/                     # doc-writing, mimir-conventions, security-checklist, monitoring
├── helm/                      # Helm values 파일
├── prometheus/                # Prometheus remote_write 설정
└── *-guide.md                 # 주제별 가이드 문서
```

---

## 커스텀 슬래시 명령어

| 명령어 | 설명 | 사용 예시 |
|--------|------|---------|
| `/new-doc` | 새 가이드 문서 생성 | `/new-doc ruler-alertmanager-integration` |
| `/new-runbook` | 새 런북 생성 | `/new-runbook Mimir Ingester 장애 대응` |
| `/review-doc` | 문서 검토 | `/review-doc multi-tenancy-guide.md` |
| `/add-troubleshooting` | 트러블슈팅 케이스 추가 | `/add-troubleshooting 수집 제한 초과` |
| `/search-kb` | 지식베이스 검색 | `/search-kb Mimir 카디널리티 제한` |

---

## 가이드 문서 목록

| 문서 | 주제 |
|------|------|
| `install.md` | Mimir 설치 (mimir-distributed Helm) |
| `architecture-guide.md` | Mimir 아키텍처 (컴포넌트별 역할) |
| `multi-tenancy-guide.md` | 멀티 테넌시 설정 |
| `remote-write-guide.md` | Prometheus remote_write 연동 |
| `recording-rules-guide.md` | Recording Rule 설정 |
| `alerting-guide.md` | Ruler + Alertmanager 알림 |
| `storage-guide.md` | S3 스토리지 구성 |
| `compactor-guide.md` | Compactor 운영 |
| `grafana-datasource-guide.md` | Grafana 데이터소스 설정 |
| `ha-guide.md` | 고가용성 구성 |
| `monitoring-guide.md` | Mimir 자체 모니터링 |
| `troubleshooting-guide.md` | 트러블슈팅 |
| `e2e-practice.md` | 엔드투엔드 실습 |
| `gitops-guide.md` | GitOps 기반 운영 |

---

## 핵심 명령어

```bash
# Mimir 컴포넌트 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=mimir

# Ingester 링 상태
curl http://mimir-ingester:8080/ring

# remote_write 테스트
curl -X POST http://mimir-distributor:8080/api/v1/push \
  -H "X-Scope-OrgID: my-tenant" \
  --data-binary @/tmp/metrics.pb

# 테넌트 카디널리티 확인
curl http://mimir-querier:8080/api/v1/cardinality/label_names \
  -H "X-Scope-OrgID: my-tenant"
```
