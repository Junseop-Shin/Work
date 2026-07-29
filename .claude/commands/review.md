# /review — Parallel Multi-Agent Code Review

Analyze changed files using parallel specialist agents and produce a unified review report.

## Steps

1. **Identify changed files and detect types**
   ```bash
   git diff --name-only HEAD
   ```
   - Frontend files: `.tsx`, `.jsx`, `.css`, `.scss`, `.vue`
   - Backend files: `.java`, `.py`, `.go`, `.sql`, server-side `.ts`/`.js`

2. **Launch specialist agents in parallel** via the Task tool:

   Always run (in parallel):
   - `code-reviewer` — bugs, logic errors, code quality, performance
   - `security-auditor` — OWASP Top 10, vulnerabilities, CWE IDs
   - `test-engineer` — test coverage gaps, missing test cases

   Conditionally (based on detected file types):
   - `frontend-developer` — if frontend files detected (UX, accessibility, React patterns)
   - `backend-developer` — if backend files detected (API design, DB patterns, N+1)

   Each agent receives: list of changed files + their full content

3. **Aggregate results** once all agents complete:

   Merge all findings into a single unified report, deduplicating overlapping issues. Tag each issue with the source agent.

4. **Output unified report** in this format:

   ```markdown
   ## Code Review Report — <date>

   ### Summary
   Files reviewed: X | Agents: Y | Issues: Z (Critical: A, Warning: B, Info: C)

   ---

   ### 🔴 Critical Issues
   - **[SEC]** `file.ts:42` — SQL injection via string concatenation
     > security-auditor · CWE-89 · CVSS 9.1
     ```diff
     - const q = `SELECT * FROM users WHERE id = ${id}`
     + const q = 'SELECT * FROM users WHERE id = ?'
     ```

   ### 🟡 Warnings
   - **[BUG]** `file.ts:88` — Missing null check before `user.profile.avatar`
     > code-reviewer

   ### 🔵 Info
   - **[TEST]** `auth.service.ts` — No test for expired token edge case
     > test-engineer

   ### ✅ Looks Good
   - ...
   ```

5. **Do not auto-fix** — report only. Apply fixes on explicit user request.

6. **워크플로우 플래그 삭제** — 리뷰 결과를 전달한 뒤 다음 작업을 위해 초기화:

```bash
rm -f ~/.claude/workflow-approved
```

그리고 알려줘: "🔄 워크플로우 플래그 초기화됨 — 다음 작업은 /plan부터 시작하세요."

## Rules

- Always run all three core agents (code-reviewer, security-auditor, test-engineer)
- Add frontend-developer / backend-developer based on actual file types — do not guess
- Deduplicate: if two agents flag the same issue, merge into one entry and note both sources
- Prioritize: Critical > Warning > Info
- Every issue must include file path and line number
