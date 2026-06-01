# mimir-practice

EKS + Grafana Mimir Distributed 기준으로 장기 메트릭 저장소, remote write, 멀티 테넌시, S3 스토리지, 규칙/알림, 고가용성을 정리한 개인 학습 문서입니다.

## 빠른 시작

- 처음 볼 문서: `docs/install.md`
- 설치 방식: Helm / systemd / Docker Compose
- 전체 흐름: 설치 -> 아키텍처/스토리지 -> 멀티 테넌시 -> remote write/read -> 규칙/알림 -> 운영
- AI 작업 지침: `CLAUDE.md`

## 구조

```text
mimir-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── README.md
│   ├── agents/
│   ├── rules/
│   ├── templates/
│   └── *.md
└── ops/
    ├── README.md
    └── config/     # Mimir Helm values와 Prometheus remote_write 예시
```

## 학습 경로

| 단계 | 문서 |
|------|------|
| 설치 | `docs/install.md` |
| 핵심 개념 | `docs/architecture-guide.md`, `docs/storage-guide.md`, `docs/multi-tenancy-guide.md` |
| 데이터 흐름 | `docs/remote-write-guide.md`, `docs/grafana-datasource-guide.md` |
| 규칙/알림 | `docs/recording-rules-guide.md`, `docs/alerting-guide.md` |
| 운영 | `docs/compactor-guide.md`, `docs/ha-guide.md`, `docs/monitoring-guide.md`, `docs/gitops-guide.md` |
| 문제 해결 | `docs/troubleshooting-guide.md`, `docs/e2e-practice.md` |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Chart | `grafana/mimir-distributed` |
| Namespace | `monitoring` |
| Storage | S3 + IRSA |
| Region | `ap-northeast-2` |
