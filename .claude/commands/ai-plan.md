The user wants a second opinion on the current plan from Gemini before proceeding.

## Steps

1. Look at the plan you just presented in this conversation (or the topic from $ARGUMENTS if provided).

2. Run Gemini with the same planning topic via Bash:

```bash
gemini "You are a senior software architect. Create a detailed implementation plan for: [topic from context]

Include:
1. Architecture overview and component breakdown
2. Technology stack with reasoning
3. Data models / API design if applicable
4. Implementation phases ordered by priority
5. Potential risks and mitigations
6. Complexity estimate per phase (S/M/L)

Be concrete and actionable."
```

3. Present a clear comparison:

**[ Claude's Plan ]** — summarize key points of your original plan
**[ Gemini's Plan ]** — summarize Gemini's response

**Differences & Notable Points** — highlight where they disagree or where Gemini adds something you missed

**Recommendation** — your final take after seeing Gemini's input

Ask the user: "어떤 방향으로 진행할까요?"

4. **워크플로우 승인 플래그 생성** — 유저가 방향을 확정하면 즉시 실행:

```bash
touch ~/.claude/workflow-approved
```

그리고 알려줘: "✅ 플랜 승인됨 — 이제 코드 작성 가능합니다. (test-engineer → implement → /ai-review 순서로 진행)"

> 이 플래그는 세션 종료 후 `/ai-review` 완료 시 자동 삭제됩니다.
