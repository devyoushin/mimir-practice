# Alerting Rules & Alertmanager

Mimir의 Ruler가 Alerting Rules를 평가하고 Alertmanager로 알림을 전송합니다.

---

## 전체 흐름

```
Ruler (규칙 평가, 1분 주기)
    │ 조건 만족 + for 시간 경과
    ▼
Alertmanager
    │ 중복 제거 → 그루핑 → 라우팅
    ▼
Slack / PagerDuty / Email / Webhook
```

### Alert 상태 주기

```
INACTIVE ──(조건 만족)──▶ PENDING ──(for 시간 경과)──▶ FIRING
    ▲                                                      │
    └──────────────────(조건 해소)─────────────────────────┘
```

- **INACTIVE**: 조건 불만족
- **PENDING**: 조건 만족하였으나 `for` 시간 미충족 (일시적 스파이크 무시)
- **FIRING**: `for` 시간 초과 → Alertmanager에 전송

---

## 1. Alerting Rules 작성

```yaml
# alerting-rules.yaml
groups:
  # ─────────────────────────────────────────────
  # 서비스 가용성
  # ─────────────────────────────────────────────
  - name: service.availability
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "서비스 {{ $labels.job }} 응답 없음"
          description: |
            {{ $labels.job }}/{{ $labels.instance }} 가 2분 이상 응답하지 않습니다.
            scrape 대상: {{ $labels.instance }}
          runbook: "https://wiki.company.com/runbooks/service-down"

      - alert: HighErrorRate
        expr: |
          (
            sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum by (job) (rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "{{ $labels.job }} 에러율 {{ $value | humanizePercentage }} 초과"
          description: |
            {{ $labels.job }}의 5분 에러율이 {{ $value | humanizePercentage }} 입니다.
            (임계값: 5%)

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1.0
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "{{ $labels.job }} P99 레이턴시 {{ $value | humanizeDuration }} 초과"
          description: "P99 레이턴시가 1초를 초과했습니다."

  # ─────────────────────────────────────────────
  # Kubernetes 리소스
  # ─────────────────────────────────────────────
  - name: kubernetes.resources
    rules:
      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "파드 {{ $labels.namespace }}/{{ $labels.pod }} 재시작 반복"
          description: "15분 내 {{ $value | printf \"%.0f\" }}번 재시작되었습니다."

      - alert: HighMemoryUsage
        expr: |
          (
            container_memory_working_set_bytes{container!=""}
            /
            container_spec_memory_limit_bytes{container!=""}
          ) > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.namespace }}/{{ $labels.pod }} 메모리 사용량 90% 초과"
          description: "현재 메모리 사용률: {{ $value | humanizePercentage }}"

      - alert: PersistentVolumeAlmostFull
        expr: |
          (
            kubelet_volume_stats_used_bytes
            /
            kubelet_volume_stats_capacity_bytes
          ) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} 용량 85% 초과"
          description: "사용률: {{ $value | humanizePercentage }}"

  # ─────────────────────────────────────────────
  # Mimir 자체 모니터링
  # ─────────────────────────────────────────────
  - name: mimir.health
    rules:
      - alert: MimirIngestionRateLimit
        expr: |
          rate(cortex_discarded_samples_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
          component: mimir
        annotations:
          summary: "Mimir ingestion rate limit 초과로 샘플 유실"
          description: |
            테넌트 {{ $labels.user }}: 초당 {{ $value | humanize }} 샘플이 거부되고 있습니다.
            limits.ingestion_rate 값을 확인하세요.

      - alert: MimirCompactorNotRunning
        expr: |
          time() - cortex_compactor_last_successful_run_timestamp_seconds > 3600
        for: 0m
        labels:
          severity: warning
          component: mimir
        annotations:
          summary: "Mimir Compactor가 1시간 이상 실행되지 않음"
          description: "마지막 성공 실행: {{ $value | humanizeDuration }} 전"
```

---

## 2. Alerting Rules 업로드

```bash
kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring

# 업로드 (네임스페이스: alerts)
curl -X POST \
  http://localhost:8080/prometheus/config/v1/rules/alerts \
  -H "X-Scope-OrgID: default" \
  -H "Content-Type: application/yaml" \
  --data-binary @alerting-rules.yaml

# mimirtool 사용 (권장)
export MIMIR_ADDRESS=http://localhost:8080
export MIMIR_TENANT_ID=default
mimirtool rules load alerting-rules.yaml
```

---

## 3. Alertmanager 설정

### Slack + PagerDuty 라우팅 예시

```yaml
# alertmanager-config.yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/XXXXX/YYYYY/ZZZZZ'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  receiver: slack-default
  group_by: ['alertname', 'namespace', 'job']
  group_wait: 30s         # 첫 알림 대기 (같은 그룹으로 묶기 위해)
  group_interval: 5m      # 그룹에 새 알림 추가 시 대기
  repeat_interval: 12h    # 같은 알림 반복 전송 간격

  routes:
    # Critical → PagerDuty + Slack 동시 알림
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: true    # 다음 route도 계속 평가

    - match:
        severity: critical
      receiver: slack-critical
      repeat_interval: 1h   # Critical은 1시간마다 반복

    # Warning → Slack만
    - match:
        severity: warning
      receiver: slack-warning

    # Mimir 내부 알림 → 인프라팀 채널
    - match:
        component: mimir
      receiver: slack-infra

receivers:
  - name: slack-default
    slack_configs:
      - channel: '#alerts'
        title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *요약:* {{ .Annotations.summary }}
          *상세:* {{ .Annotations.description }}
          {{ if .Annotations.runbook }}*런북:* {{ .Annotations.runbook }}{{ end }}
          {{ end }}
        send_resolved: true

  - name: slack-critical
    slack_configs:
      - channel: '#alerts-critical'
        color: 'danger'
        title: '[CRITICAL] {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts.Firing }}
          *요약:* {{ .Annotations.summary }}
          *상세:* {{ .Annotations.description }}
          *시작:* {{ .StartsAt | since | humanizeDuration }} 전
          {{ if .Annotations.runbook }}*런북:* <{{ .Annotations.runbook }}|바로가기>{{ end }}
          {{ end }}
        send_resolved: true

  - name: slack-warning
    slack_configs:
      - channel: '#alerts-warning'
        color: 'warning'
        title: '[WARNING] {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: slack-infra
    slack_configs:
      - channel: '#infra-alerts'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: '<PD_INTEGRATION_KEY>'
        description: '{{ .GroupLabels.alertname }}: {{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        severity: critical
        details:
          firing: '{{ .Alerts.Firing | len }}'
          description: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

# Inhibition: Critical이 발생하면 동일 job의 Warning 억제
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'namespace', 'job']
```

### Alertmanager 설정 업로드

```bash
# Alertmanager API로 설정 업로드
curl -X POST \
  http://localhost:8080/alertmanager/api/v1/alerts \
  -H "X-Scope-OrgID: default" \
  -H "Content-Type: application/yaml" \
  --data-binary @alertmanager-config.yaml

# mimirtool 사용 (권장)
mimirtool alertmanager load alertmanager-config.yaml

# 현재 설정 확인
mimirtool alertmanager print
```

---

## 4. 알림 상태 확인

```bash
# 현재 발생 중인 알림 목록
curl http://localhost:8080/prometheus/api/v1/alerts \
  -H "X-Scope-OrgID: default" | \
  jq '.data.alerts[] | {name: .labels.alertname, state: .state, severity: .labels.severity}'

# FIRING 알림만
curl http://localhost:8080/prometheus/api/v1/alerts \
  -H "X-Scope-OrgID: default" | \
  jq '.data.alerts[] | select(.state == "firing")'

# Alertmanager UI 접근
kubectl port-forward svc/mimir-alertmanager 9093:9093 -n monitoring
# 브라우저: http://localhost:9093

# Alertmanager API로 활성 알림 조회
curl http://localhost:9093/api/v2/alerts | jq
```

---

## 5. Silence (음소거)

유지보수 시 알림을 일시적으로 음소거합니다:

```bash
# 특정 알림 음소거 (2시간)
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "alertname", "value": "PodCrashLooping", "isRegex": false}
    ],
    "startsAt": "2026-04-04T00:00:00Z",
    "endsAt": "2026-04-04T02:00:00Z",
    "comment": "배포 작업으로 인한 임시 음소거",
    "createdBy": "operator"
  }'

# 전체 네임스페이스 음소거
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "namespace", "value": "staging", "isRegex": false}
    ],
    "startsAt": "2026-04-04T00:00:00Z",
    "endsAt": "2026-04-04T04:00:00Z",
    "comment": "스테이징 유지보수",
    "createdBy": "operator"
  }'

# 음소거 목록 확인
curl http://localhost:9093/api/v2/silences | jq '.[].comment'
```

---

## 트러블슈팅

```bash
# Ruler 로그 확인 (규칙 평가 오류)
kubectl logs -n monitoring -l app.kubernetes.io/component=ruler -f \
  | grep -i "error\|fail"

# 규칙 평가 오류 확인
curl http://localhost:8080/prometheus/api/v1/rules \
  -H "X-Scope-OrgID: default" | \
  jq '.data.groups[].rules[] | select(.health != "ok") | {name: .name, health: .health, lastError: .lastError}'

# Alertmanager 로그 (알림 발송 오류)
kubectl logs -n monitoring -l app.kubernetes.io/component=alertmanager -f \
  | grep -i "error\|slack\|pagerduty"
```

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| 알림이 발생하지 않음 | 규칙 문법 오류 | `mimirtool rules lint` 실행 |
| PENDING에서 FIRING 안됨 | `for` 시간 미충족 | 조건이 실제 만족인지 재확인 |
| Slack 전송 안됨 | Webhook URL 만료 | Slack App에서 새 Webhook 발급 |
| 중복 알림 과다 | `repeat_interval` 짧음 | `repeat_interval: 12h` 이상으로 조정 |
| FIRING 후 즉시 해소됨 | 조건이 불안정 | `for: 5m` 이상으로 안정화 |
