Run a multi-AI code review using cmux + 3-pane layout.

**전제 조건**: cmux 터미널 안에서 Claude를 실행해야 합니다 (`CMUX_WORKSPACE_ID` 필요).

## Steps

1. Get the changes:
```bash
git diff HEAD
git status --short
```

2. Write a context file:
```bash
CONTEXT_FILE="/tmp/ai-review-$(date +%H%M%S).md"
```

Write to that file:
```
# Multi-AI Code Review

## Git Diff

[paste actual git diff output here]
```

3. Launch the review:
```bash
ai-review-launch "$CONTEXT_FILE"
```

## What happens next

- cmux가 새 워크스페이스(세션)를 생성합니다
- 3-pane 레이아웃으로 분할됩니다:
  - **왼쪽 (Coordinator)**: 진행 상황 표시 + Claude 취합
  - **오른쪽 위 (Gemini)**: Gemini 리뷰 실시간 출력
  - **오른쪽 아래 (Codex)**: Codex 리뷰 실시간 출력
- 완료되면 원래 세션으로 결과(`less /tmp/ai-review-latest.md`)를 전달합니다
- 리뷰 세션은 자동으로 종료됩니다

4. Tell the user: "새 cmux 워크스페이스에서 AI 리뷰를 시작했습니다. 완료되면 자동으로 결과가 전달됩니다."

5. **워크플로우 플래그 삭제** — 리뷰 완료 후 다음 작업을 위해 초기화:

```bash
rm -f ~/.claude/workflow-approved
```

그리고 알려줘: "🔄 워크플로우 플래그 초기화됨 — 다음 작업은 /ai-plan부터 시작하세요."
