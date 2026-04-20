Mimir 가이드 문서를 검토합니다.

**사용법**: `/review-doc <파일 경로>`  **예시**: `/review-doc multi-tenancy-guide.md`

검토 기준:
- 멀티 테넌시: X-Scope-OrgID 헤더 설정, 테넌트 격리
- 스토리지: S3 설정 (IRSA), 압축 정책
- HA: 인게스터 replication factor, 샤딩 설정
- 비용: 컴팩션 주기, 데이터 보존 기간
- 문서: API 예시, PromQL 쿼리 포함
