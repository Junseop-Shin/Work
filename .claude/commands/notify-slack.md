# /notify-slack — Slack 알림 전송

작업 완료, 승인 요청, 빌드 결과 등을 Slack으로 전송한다.

## Steps

1. **메시지 구성** — `$ARGUMENTS` 가 있으면 그대로 사용. 없으면 현재 컨텍스트에서 자동 생성:
   - 직전에 완료한 작업 요약 (1-3줄)
   - 승인이 필요하면 "✅ 승인 필요: [무엇을]" 형태로
   - 빌드/배포 결과면 성공/실패 + 링크 포함

2. **Webhook URL 로드**
   ```bash
   # $SLACK_WEBHOOK_URL 환경변수가 설정되어 있어야 함 (secrets/credentials.md 참고)
   # 미설정 시 devops-monitor .env에서 수동으로 export 후 실행
   ```

3. **전송**
   ```bash
   curl -s -X POST "$SLACK_WEBHOOK_URL" \
     -H 'Content-Type: application/json' \
     -d "{\"text\": \"$MESSAGE\"}"
   ```

4. **결과 확인** — curl 응답이 `ok` 면 성공, 아니면 오류 출력

## 메시지 포맷 가이드

| 상황 | 포맷 |
|------|------|
| 작업 완료 | `✅ [프로젝트] 작업 완료: <내용>` |
| 승인 요청 | `🔔 [프로젝트] 승인 필요: <무엇을 승인해야 하는지>` |
| 빌드 완료 | `📦 [프로젝트] 빌드 완료: <버전> — <다운로드 링크>` |
| 배포 완료 | `🚀 [프로젝트] 배포 완료: <환경> <URL>` |
| 오류 발생 | `🚨 [프로젝트] 오류: <내용>` |

## Rules
- webhook URL은 항상 devops-monitor `.env`에서 읽음 — 메시지에 URL 노출 금지
- `$ARGUMENTS` 로 커스텀 메시지 전달 가능: `/notify-slack 배포 완료`
- 멘션이 필요하면 `<!channel>` 또는 `<@USERID>` 사용 (사용자가 명시한 경우만)
