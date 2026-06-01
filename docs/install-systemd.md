# Mimir systemd 설치

단일 VM이나 베어메탈에서 Mimir를 서비스로 관리할 때 사용한다.

## 대상

- `mimir.service`
- `mimir-gateway.service`

## 준비물

- Mimir 바이너리
- 설정 파일: `/etc/mimir/mimir.yaml`
- 데이터 디렉터리: `/var/lib/mimir`

## 절차

1. 바이너리를 설치한다.
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
