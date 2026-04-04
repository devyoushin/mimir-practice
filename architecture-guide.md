# Mimir 아키텍처

---

## 전체 구조

```
                  ┌──────────────────────────────────────────────────────┐
                  │                   Mimir Cluster                      │
                  │                                                      │
Prometheus ──────▶│  Distributor ──▶ Ingester (WAL + mem) ──▶ S3 (TSDB)│
(remote_write)    │       │               │                      │      │
X-Scope-OrgID     │  consistent      replication                 │      │
                  │    hashing        factor=3             Store Gateway │◀── Grafana
                  │                                              │      │
                  │  Query Frontend ◀──── Querier ◀─────────────┘      │
                  │  (캐싱/샤딩)              └── Ingester (최근 데이터) │
                  │                                                      │
                  │  Ruler ──▶ Recording/Alerting Rules                 │
                  │  Alertmanager ──▶ Slack / PagerDuty / Email         │
                  │  Compactor ──▶ 블록 병합 + 만료 삭제                 │
                  └──────────────────────────────────────────────────────┘
```

---

## 컴포넌트 상세

### Distributor (쓰기 경로 입구)

- Prometheus `remote_write` 요청을 수신
- 샘플의 유효성 검증 (레이블 이름, UTF-8, 메트릭 이름 형식)
- 일관된 해싱(consistent hashing)으로 Ingester에 분산
- `replication_factor`에 따라 여러 Ingester에 동시 복제
- **Stateless** — 자유롭게 수평 확장 가능

```
요청 수신 → 유효성 검증 → 해시 링에서 Ingester 선택 → replication_factor만큼 동시 전송
```

**Write Quorum**: `ceil(replication_factor / 2)` 개 이상의 Ingester가 성공 응답해야 수락
→ `replication_factor=3` 이면 최소 2개 성공 필요

---

### Ingester (인메모리 버퍼 + WAL)

- 수신한 샘플을 **메모리에 TSDB 형태**로 보관
- **WAL (Write-Ahead Log)**: 파드 재시작 시 메모리 데이터 복구용
  - PersistentVolume에 저장 (`/data/wal`)
  - 재시작 시 WAL을 replay하여 메모리 복원
- 기본 **2시간마다** S3에 TSDB 블록으로 플러시
- 플러시 후에도 `blocks_storage.tsdb.retention_period` 동안 메모리 유지 (중복 쿼리 방지)
- **StatefulSet** — 파드 재시작 순서 보장 + PVC 유지

```
샘플 수신 → WAL 기록 → 메모리 TSDB 저장 → 2h마다 S3 플러시
                ↑
         파드 재시작 시 여기서 복구
```

---

### Query Frontend (쿼리 최적화 레이어)

- 클라이언트(Grafana)의 쿼리 요청 수신
- **긴 범위 쿼리를 시간 단위로 분할**하여 Querier에 병렬 전송
- **결과 캐싱**: Memcached 또는 인메모리 캐시
- **쿼리 샤딩**: 고카디널리티 집계 쿼리를 여러 Querier에 분산
- 동시 요청 수 제한 및 재시도 처리
- **Stateless** — 자유롭게 수평 확장

---

### Querier (실제 쿼리 실행)

- Query Frontend에서 받은 PromQL을 실행
- **두 가지 소스를 합산하여 반환**:

```
Querier
  ├── Ingester   → 최근 데이터 (메모리, 아직 S3에 없는 데이터)
  └── Store Gateway → S3 블록 (장기 저장 데이터)
```

- 중복 데이터(Ingester와 S3 모두에 있는 구간)는 자동 중복 제거
- **Stateless** — 자유롭게 수평 확장

---

### Store Gateway (S3 블록 인덱스 캐시)

- S3에 저장된 TSDB 블록의 **인덱스를 로컬 PVC에 캐싱**
- Querier가 S3에 직접 매번 접근하지 않도록 메타데이터 제공
- Zone-aware sharding으로 블록을 파티셔닝하여 파드 간 분산
- **StatefulSet** — PVC에 인덱스 캐시 유지 (재시작 시 재다운로드 방지)

---

### Compactor (블록 최적화)

- S3의 소규모 블록들을 주기적으로 **병합 (compaction)**
- `2h → 12h → 24h → 48h` 계층적 병합
- 만료된 블록 삭제 (retention 정책 적용)
- 수평 확장 불필요 — **단일 파드**로 충분 (sharding_enabled로 확장 가능)

---

### Ruler (규칙 엔진)

- **Recording Rules**: 복잡한 쿼리를 주기적으로 실행하여 새 메트릭 생성
- **Alerting Rules**: 조건 만족 시 Alertmanager에 알림 전송
- Mimir에 저장된 규칙 파일을 Ruler API로 관리
- 규칙은 **테넌트별로 격리**됨

---

### Alertmanager (알림 관리)

- Ruler에서 전달된 알림을 수신
- 중복 제거, 그루핑, 라우팅, 음소거(silencing)
- Slack / Email / PagerDuty 등으로 발송
- 테넌트별 알림 설정 지원
- **HA 모드**: 3개 복제로 알림 유실 방지

---

## 쓰기 경로 (Write Path)

```
Prometheus
    │ HTTP POST (Snappy 압축 protobuf)
    │ X-Scope-OrgID: tenant-a
    ▼
mimir-nginx (Gateway)
    │
    ▼
Distributor
    │ consistent hash(metric + labels)
    │ replication_factor=3
    ├──▶ Ingester-0 (AZ-a)  ──┐
    ├──▶ Ingester-1 (AZ-b)  ──┤ 최소 2개 성공해야 200 OK
    └──▶ Ingester-2 (AZ-c)  ──┘
              │
              │ 2시간 경과 시 자동 플러시
              ▼
          S3 (TSDB 블록)
          <tenant-id>/
            <ULID>/
              chunks/
              index
              meta.json
              tombstones
```

---

## 읽기 경로 (Read Path)

```
Grafana / PromQL 클라이언트
    │ HTTP GET
    │ X-Scope-OrgID: tenant-a
    ▼
mimir-nginx (Gateway)
    │
    ▼
Query Frontend
    │ 시간 범위 분할 (예: 7일 쿼리 → 7개 1일 단위로 분할)
    │ 쿼리 샤딩 (label matcher 기준)
    ▼
Querier (병렬 실행)
    │
    ├──▶ Ingester     → 최근 2~3시간 데이터 (메모리)
    │
    └──▶ Store Gateway → S3 블록 메타데이터/인덱스 → S3 실제 데이터

결과 병합 + 중복 제거 → Query Frontend 캐시 → Grafana 반환
```

---

## 데이터 생명주기

```
[0h]   Prometheus scrape → remote_write → Distributor → Ingester 메모리 + WAL
[2h]   Ingester → S3 블록 플러시 (Level 1: 2h 블록)
[12h]  Compactor: 6개 × 2h 블록 → 1개 × 12h 블록 (Level 2)
[24h]  Compactor: 2개 × 12h 블록 → 1개 × 24h 블록 (Level 3)
[48h]  Compactor: 2개 × 24h 블록 → 1개 × 48h 블록 (Level 4)
[90d]  Compactor: retention 만료 → tombstone 마킹 → 삭제
```

---

## 링 (Ring) 아키텍처

Mimir는 **분산 해시 링**으로 Ingester / Store Gateway / Compactor를 조율합니다.

```
Ring (KV Store: memberlist 또는 etcd)

Token 0         Token 100       Token 200       Token 300 ...
  │               │               │               │
Ingester-0      Ingester-1      Ingester-2      Ingester-0
(AZ-a)          (AZ-b)          (AZ-c)
```

- 각 Ingester는 링에 **토큰**을 등록
- Distributor는 `hash(metric + labels)` 로 링에서 담당 Ingester 선택
- Zone-aware replication: 각 AZ에서 1개씩 선택하여 복제

---

## 참고 링크

- [Mimir 아키텍처 공식 문서](https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/)
- [Mimir 설계 원칙](https://grafana.com/docs/mimir/latest/get-started/why-use-mimir/)
- [Consistent Hashing / Ring 설명](https://grafana.com/docs/mimir/latest/operators-guide/architecture/hash-ring/)
