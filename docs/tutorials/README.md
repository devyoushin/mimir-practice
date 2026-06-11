# 실습 문서

Prometheus에서 Mimir로 메트릭을 전송하고 Grafana에서 조회하는 End-to-End 실습 문서를 모은 폴더입니다.

## 문서 목록

| 문서 | 용도 |
|------|------|
| [e2e-practice.md](e2e-practice.md) | Prometheus -> Mimir -> Grafana 전체 흐름 검증 |

## 사전 학습

1. [../install/install.md](../install/install.md)
2. [../storage/storage-guide.md](../storage/storage-guide.md)
3. [../ingestion/remote-write-guide.md](../ingestion/remote-write-guide.md)

## 관련 경로

- [../../ops/config/helm/values.yaml](../../ops/config/helm/values.yaml)
- [../../ops/config/prometheus/remote-write.yaml](../../ops/config/prometheus/remote-write.yaml)
