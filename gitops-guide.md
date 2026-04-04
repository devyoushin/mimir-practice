# GitOps로 Mimir Rules 관리

Recording Rules / Alerting Rules / Alertmanager 설정을 Git으로 버전 관리하고
CI/CD로 자동 배포하는 방법입니다.

---

## 왜 GitOps인가?

| 수동 관리 문제 | GitOps 해결책 |
|---|---|
| 누가 언제 규칙을 변경했는지 모름 | Git 커밋 히스토리로 추적 |
| 테스트 없이 운영에 직접 적용 | PR 리뷰 + CI lint 통과 후 병합 |
| 클러스터 재설치 시 규칙 유실 | Git이 단일 진실의 원천(Source of Truth) |
| 롤백이 어려움 | `git revert`로 즉시 롤백 |

---

## 디렉토리 구조 권장 설계

```
mimir-practice/
└── rules/
    ├── default/                    # 테넌트: default
    │   ├── recording/
    │   │   ├── http.yaml
    │   │   ├── kubernetes.yaml
    │   │   └── infrastructure.yaml
    │   └── alerting/
    │       ├── service.yaml
    │       ├── kubernetes.yaml
    │       └── mimir-health.yaml
    ├── team-a/                     # 테넌트: team-a
    │   ├── recording/
    │   └── alerting/
    └── alertmanager/
        ├── default.yaml            # 테넌트별 Alertmanager 설정
        └── team-a.yaml
```

---

## 1. 규칙 파일 작성 예시

```yaml
# rules/default/recording/http.yaml
groups:
  - name: http:red
    interval: 1m
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))
```

---

## 2. Makefile (로컬 개발 편의)

```makefile
# Makefile
MIMIR_ADDRESS ?= http://localhost:8080
TENANT ?= default

.PHONY: port-forward lint diff sync sync-all

# Mimir 포트 포워드
port-forward:
	kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring &

# 문법 검사
lint:
	mimirtool rules lint rules/$(TENANT)/recording/*.yaml rules/$(TENANT)/alerting/*.yaml

# 서버와 diff 비교
diff:
	MIMIR_ADDRESS=$(MIMIR_ADDRESS) MIMIR_TENANT_ID=$(TENANT) \
	  mimirtool rules diff rules/$(TENANT)/recording/*.yaml rules/$(TENANT)/alerting/*.yaml

# 특정 테넌트 동기화
sync:
	MIMIR_ADDRESS=$(MIMIR_ADDRESS) MIMIR_TENANT_ID=$(TENANT) \
	  mimirtool rules sync rules/$(TENANT)/recording/*.yaml rules/$(TENANT)/alerting/*.yaml

# 모든 테넌트 동기화
sync-all:
	for tenant in rules/*/; do \
	  tenant_name=$$(basename $$tenant); \
	  echo "Syncing tenant: $$tenant_name"; \
	  MIMIR_ADDRESS=$(MIMIR_ADDRESS) MIMIR_TENANT_ID=$$tenant_name \
	    mimirtool rules sync $$tenant/**/*.yaml; \
	done

# Alertmanager 설정 배포
deploy-alertmanager:
	MIMIR_ADDRESS=$(MIMIR_ADDRESS) MIMIR_TENANT_ID=$(TENANT) \
	  mimirtool alertmanager load rules/alertmanager/$(TENANT).yaml
```

---

## 3. GitHub Actions CI/CD

### PR 시 검증 (Lint + Diff)

```yaml
# .github/workflows/validate-rules.yaml
name: Validate Mimir Rules

on:
  pull_request:
    paths:
      - 'rules/**'
      - 'alertmanager/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install mimirtool
        run: |
          VERSION=$(curl -s https://api.github.com/repos/grafana/mimir/releases/latest | jq -r '.tag_name')
          curl -Lo mimirtool \
            "https://github.com/grafana/mimir/releases/download/${VERSION}/mimirtool-linux-amd64"
          chmod +x mimirtool
          sudo mv mimirtool /usr/local/bin/

      - name: Lint all rules
        run: |
          find rules -name "*.yaml" -exec mimirtool rules lint {} +

      - name: Check rule naming convention
        run: |
          # recording rule 이름이 level:metric:operation 형식인지 검사
          python3 scripts/check-naming.py rules/

      - name: Validate alerting rules
        run: |
          # 모든 alert에 severity 레이블이 있는지 확인
          find rules -path "*/alerting/*.yaml" | while read f; do
            if ! grep -q "severity:" "$f"; then
              echo "ERROR: $f is missing severity label"
              exit 1
            fi
          done
```

### main 병합 시 자동 배포

```yaml
# .github/workflows/deploy-rules.yaml
name: Deploy Mimir Rules

on:
  push:
    branches: [main]
    paths:
      - 'rules/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production    # GitHub Environments로 승인 필요 시
    steps:
      - uses: actions/checkout@v4

      - name: Install mimirtool
        run: |
          curl -Lo mimirtool \
            https://github.com/grafana/mimir/releases/latest/download/mimirtool-linux-amd64
          chmod +x mimirtool

      - name: Configure kubectl (EKS)
        run: |
          aws eks update-kubeconfig \
            --region ap-northeast-2 \
            --name ${{ secrets.EKS_CLUSTER_NAME }}

      - name: Port forward Mimir
        run: |
          kubectl port-forward svc/mimir-nginx 8080:80 -n monitoring &
          sleep 5
          curl -s http://localhost:8080/ready

      - name: Sync rules (default tenant)
        env:
          MIMIR_ADDRESS: http://localhost:8080
          MIMIR_TENANT_ID: default
        run: |
          mimirtool rules sync \
            rules/default/recording/*.yaml \
            rules/default/alerting/*.yaml

      - name: Sync rules (team-a tenant)
        env:
          MIMIR_ADDRESS: http://localhost:8080
          MIMIR_TENANT_ID: team-a
        run: |
          mimirtool rules sync \
            rules/team-a/recording/*.yaml \
            rules/team-a/alerting/*.yaml

      - name: Verify deployment
        run: |
          # 배포 후 규칙이 정상 평가되는지 확인
          sleep 30
          ERRORS=$(curl -s http://localhost:8080/prometheus/api/v1/rules \
            -H "X-Scope-OrgID: default" | \
            jq '[.data.groups[].rules[] | select(.health != "ok")] | length')
          if [ "$ERRORS" -gt 0 ]; then
            echo "ERROR: ${ERRORS} rules are not healthy"
            exit 1
          fi
          echo "All rules are healthy"

      - name: Notify Slack on failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H "Content-Type: application/json" \
            -d '{"text": "❌ Mimir rules 배포 실패! GitHub Actions를 확인하세요."}'
```

---

## 4. ArgoCD로 관리 (GitOps 오퍼레이터)

Helm Chart와 규칙을 함께 ArgoCD로 관리:

```yaml
# argocd-mimir-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mimir
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mycompany/mimir-practice
    targetRevision: main
    path: helm
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 5. 규칙 롤백

```bash
# Git으로 이전 버전 확인
git log --oneline rules/default/alerting/service.yaml

# 특정 커밋으로 롤백
git revert <commit-hash>
git push origin main
# → CI/CD가 자동으로 이전 규칙 재배포

# 또는 직접 이전 버전 체크아웃
git checkout <commit-hash> -- rules/default/alerting/service.yaml
git commit -m "revert: rollback service alerts to stable version"
git push origin main
```

---

## 6. 규칙 네이밍 컨벤션 검사 스크립트

```python
# scripts/check-naming.py
import yaml
import sys
import re
import os

RECORDING_RULE_PATTERN = re.compile(r'^[a-z_]+:[a-z_]+:[a-z0-9_]+$')

def check_file(filepath):
    errors = []
    with open(filepath) as f:
        data = yaml.safe_load(f)

    if not data or 'groups' not in data:
        return errors

    for group in data.get('groups', []):
        for rule in group.get('rules', []):
            if 'record' in rule:
                name = rule['record']
                if not RECORDING_RULE_PATTERN.match(name):
                    errors.append(
                        f"{filepath}: recording rule '{name}' "
                        f"must match pattern 'level:metric:operation'"
                    )
            if 'alert' in rule:
                labels = rule.get('labels', {})
                if 'severity' not in labels:
                    errors.append(
                        f"{filepath}: alert '{rule['alert']}' "
                        f"is missing 'severity' label"
                    )
    return errors

if __name__ == '__main__':
    rules_dir = sys.argv[1] if len(sys.argv) > 1 else 'rules'
    all_errors = []

    for root, _, files in os.walk(rules_dir):
        for fname in files:
            if fname.endswith('.yaml'):
                path = os.path.join(root, fname)
                all_errors.extend(check_file(path))

    if all_errors:
        for e in all_errors:
            print(f"ERROR: {e}")
        sys.exit(1)
    else:
        print(f"All rules passed naming convention checks.")
```

---

## 참고 링크

- [mimirtool 공식 문서](https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/)
- [mimirtool rules sync](https://grafana.com/docs/mimir/latest/manage/tools/mimirtool/#rules)
- [Grafana Mimir + ArgoCD 예시](https://grafana.com/docs/mimir/latest/operators-guide/deploy-grafana-mimir/jsonnet/)
