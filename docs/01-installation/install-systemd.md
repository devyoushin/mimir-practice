# Mimir systemd 설치

단일 VM이나 베어메탈에서 Mimir를 서비스로 관리할 때 사용한다.

## 대상

- `mimir.service`
- `mimir-gateway.service`

## 준비물

- Mimir 바이너리
- 설정 파일: `/etc/mimir/mimir.yaml`
- 데이터 디렉터리: `/var/lib/mimir`

## RPM 설치

Mimir는 단일 바이너리 배포나 컨테이너 배포를 많이 사용한다. RPM으로 운영하려면 내부 패키지 저장소나 직접 빌드한 RPM을 기준으로 설치한다.

```bash
sudo dnf install -y ./mimir-<VERSION>-1.x86_64.rpm
rpm -qa | grep mimir
rpm -ql mimir | head
```

RPM이 systemd unit을 포함하지 않는 경우에는 `/etc/systemd/system/mimir.service`를 직접 작성한다.

## tarball 설치

릴리스 바이너리를 내려받아 `/usr/local/bin/mimir`에 배치한다.

```bash
MIMIR_VERSION="<VERSION>"
curl -LO "https://github.com/grafana/mimir/releases/download/mimir-${MIMIR_VERSION}/mimir-linux-amd64"
chmod +x mimir-linux-amd64
sudo install -m 0755 mimir-linux-amd64 /usr/local/bin/mimir
sudo mkdir -p /etc/mimir /var/lib/mimir
```

다운로드 URL 형식은 릴리스 버전에 따라 달라질 수 있으므로 실제 운영에서는 릴리스 페이지의 asset 이름을 확인하고 고정한다.

## 절차

1. RPM 또는 tarball 방식 중 하나로 바이너리를 설치한다.
2. 설정 파일과 runtime directory를 만든다.
3. systemd unit 파일을 `/etc/systemd/system/`에 둔다.
4. `systemctl enable --now mimir`로 시작한다.
5. `journalctl -u mimir -f`로 로그를 본다.

## 확인 명령

```bash
systemctl status mimir
curl http://localhost:8080/ready
```

## 운영 포인트

- S3 인증은 IAM role 또는 환경 변수로 주입한다.
- ingester WAL과 cache 디렉터리는 유지해야 한다.
- 멀티 노드 구성은 Helm 문서를 우선 기준으로 삼는다.
