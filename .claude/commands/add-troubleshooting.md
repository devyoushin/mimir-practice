Mimir 트러블슈팅 케이스를 추가합니다.

**사용법**: `/add-troubleshooting <증상 설명>`  **예시**: `/add-troubleshooting 인게스터 OOM`

형식:
```markdown
### <증상>
**원인**: <근본 원인>
**확인**:
\`\`\`bash
kubectl logs -n monitoring -l app.kubernetes.io/component=ingester
curl http://mimir/api/v1/status/config
\`\`\`
**해결**: <해결 방법>
```
`troubleshooting-guide.md`에 추가하세요.
