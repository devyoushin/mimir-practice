# Mimir Docker Compose 설치

로컬이나 단일 노드에서 Mimir를 빠르게 검증할 때 사용한다.

## 대상

- `mimir`
- `mimir-nginx`

## 절차

1. `compose.yaml`에 이미지, 포트, volume을 정의한다.
2. Mimir 설정 파일과 버킷 설정을 mount 한다.
3. `docker compose up -d`로 올린다.
4. `docker compose logs -f`와 `docker compose ps`를 확인한다.

## 확인 명령

```bash
docker compose ps
curl http://localhost:8080/ready
```

## 운영 포인트

- 로컬 실험은 object storage 없이도 가능한 최소 설정으로 둔다.
- 운영용 IRSA/S3 구성은 Helm 문서를 따른다.
