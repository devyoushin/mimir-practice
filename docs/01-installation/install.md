# Mimir 설치 안내

설치 방식별 문서를 분리해 둔 입구다.

## 설치 방식

| 방식 | 문서 | 설명 |
|------|------|------|
| Helm | `install-helm.md` | EKS 기준 `mimir-distributed` 설치 |
| systemd | `install-systemd.md` | RPM 또는 tarball로 설치 후 Mimir 또는 gateway를 서비스로 관리 |
| Docker Compose | `install-docker-compose.md` | 로컬/단일 노드 테스트용 |
| Upgrade | `upgrade/` | Helm, systemd, Docker Compose 업그레이드 |

## 읽는 순서

1. `install-helm.md`
2. `install-systemd.md`
3. `install-docker-compose.md`
4. `upgrade/`
