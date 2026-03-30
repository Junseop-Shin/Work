# CLAUDE.md — AI Dev Environment Guidelines

> This file configures Claude Code behavior for `~/Documents/Work`.
> It is the **canonical source** — other AI tool configs (GEMINI.md, AGENTS.md, etc.) summarize these rules.
> Keep this file lean — review and trim periodically.

---

## Access Policy

| Path | AI Access | Purpose |
|------|-----------|---------|
| `~/Documents/General/` | **FORBIDDEN** | Personal documents, certificates, financial records |
| `~/Documents/Work/` | **ALLOWED** | Main AI workspace |

- Never read, write, or reference files outside `~/Documents/Work/`
- Never access `.env` files or files containing secrets
- Never modify CI/CD pipeline configurations without explicit approval

---

## Model Selection Rules

| Task Type | Model |
|-----------|-------|
| File search (Glob/Grep), directory traversal, quick lookups | **Haiku** |
| Bash commands, git status checks, simple reads | **Haiku** |
| Code writing/editing, debugging, refactoring, test writing | **Sonnet** |
| PR review, API design, complex logic analysis | **Sonnet** |
| Architecture design, system planning, technology decisions | **Opus** |
| Sub-agent: file-explorer | **Haiku** |
| Sub-agent: architect | **Opus** |
| Sub-agent: general-purpose, code-reviewer | **Sonnet** |

Use the cheapest model that can handle the task. Escalate only when needed.

---

## Workflow

Always follow this sequence — do not skip steps:

1. **Understand** — Read relevant files before making changes
2. **Plan** — Outline changes before executing (use EnterPlanMode for non-trivial tasks)
3. **Implement** — Make focused, minimal changes
4. **Verify** — Run tests or check output before committing
5. **Commit** — Only after explicit user approval

For complex multi-step tasks, use the Task tool to delegate to specialized sub-agents.

Break down large problems into sub-problems. Identify intermediate milestones. Prefer the simplest solution that meets requirements.

---

## Context Management

AI context degrades over long sessions — keep it fresh:

- Start a new session when switching to an unrelated task
- Use `/compact` to summarize and compress the current session
- Before ending a long session, create a handoff doc: `echo "## Handoff\n..." > handoff.md` so the next session can pick up quickly
- Run parallel work in separate terminal tabs (multiple Claude sessions)

---

## Verification Strategies

Always verify before declaring done:

- Write tests or run existing ones
- Use `gh pr create --draft` to safely review changes via GitHub UI
- Ask Claude to double-check its own output: "Review what you just wrote for bugs"
- For UI changes, use screenshots or visual diffs
- For long-running tasks, use background processes: `Ctrl+B` to background, check output later

---

## Bash Safety

- Break complex bash commands into simple sequential steps — avoid chaining with `&&` unless steps are truly dependent
- Avoid pipes (`|`) when possible — run steps individually or write intermediate results to files
- Never mix stderr and stdout: do not use `2>&1`
- Use `realpath` to get absolute file paths for reliable references
- For risky or experimental scripts, isolate in a Docker container before running on host

---

## Commit Rules

- Follow **Conventional Commits** format: `type(scope): description`
  - Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`
- **Never commit without explicit user approval**
- **Never force-push or reset --hard** without explicit instruction
- **Never commit to `main` directly** — always use feature branches
- Branch naming: `feat/`, `fix/`, `docs/`, `chore/` prefix

### Branch & PR Policy

- Create a new branch for each unit of work
- After committing, push branch and create PR via `gh pr create`
- Merge is **manual** — user reviews PR before merging
- PR title follows the same Conventional Commits format

### Commit Message Format

```
type(scope): short description (max 72 chars)

- Bullet points for details if needed
- Reference issues: Closes #123

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

---

## Language Policy

| Context | Language |
|---------|----------|
| Conversation with user | Korean OK (match user's language) |
| Code, variable names, comments | English |
| Commit messages | English |
| PR titles and descriptions | English |
| File/directory names | English |

---

## Security Rules

- Never read or output contents of `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12` files
- Never hardcode secrets, tokens, or credentials in any file
- Use environment variable references (e.g., `$GITHUB_PERSONAL_ACCESS_TOKEN`) in configs
- Never disable security checks (`--no-verify`, skipping auth) without explicit instruction
- Validate all user inputs at system boundaries
- Avoid SQL injection, XSS, command injection, and other OWASP Top 10 vulnerabilities

---

## Sub-Agent Usage

Use sub-agents via the Task tool for:

| Sub-Agent | When to Use |
|-----------|-------------|
| `file-explorer` | Searching files, grepping code, directory traversal |
| `architect` | System design, architecture decisions, tech selection |
| `code-reviewer` | Code quality review, security audit, PR analysis |

### Custom Commands

| Command | Purpose |
|---------|---------|
| `/commit` | Analyze staged changes → generate Conventional Commit → commit + push + PR |
| `/review` | Analyze changed files → generate bug/security/quality report |
| `/plan` | Invoke Opus sub-agent → produce architecture design doc |
| `/ai-plan` | Gemini second opinion on plan → comparison → user approval |
| `/ai-review` | Gemini second opinion on code review |
| `/standup` | Summarize today's commits → generate standup notes |

### Workflow Harness (반드시 순서대로 실행)

Planning → Implementation → Review 워크플로우:

1. `/plan` — 아키텍처 설계 + 전문가 리뷰
2. `/ai-plan` — Gemini 세컨드 오피니언 + 비교 + 유저 승인
3. 구현 (승인 후에만 코드 작성)
4. `/review` — 코드 리뷰 (버그/보안/품질)
5. `/ai-review` — Gemini 세컨드 오피니언 코드 리뷰

**Review Loop (이슈 0건이 될 때까지 반복):**
```
LOOP:
  /review → 이슈 발견?
    YES → 이슈 수정 → LOOP 재실행
    NO  → /ai-review → 이슈 발견?
      YES → 이슈 수정 → LOOP 재실행
      NO  → 리뷰 통과 → 커밋/배포 진행
```

**강제 규칙:**
- `/plan` 실행 후 반드시 `/ai-plan`을 이어서 실행할 것
- 구현 완료 후 반드시 `/review` → `/ai-review` 순서로 실행할 것
- 리뷰에서 이슈가 발견되면 수정 후 다시 리뷰 루프 재실행 (이슈 0건까지)
- 유저가 명시적으로 스킵하지 않는 한 생략 금지

---

## Code Quality Standards

- Avoid over-engineering — solve the problem at hand, not hypothetical futures
- No premature abstractions — three similar lines > one questionable helper
- No unnecessary error handling for impossible scenarios
- Trust framework guarantees — only validate at system boundaries
- Delete unused code completely — no backwards-compatibility shims
- No docstrings or comments for self-evident code

---

## Infrastructure Access

### Windows Home Server (SSH)

```bash
ssh windows   # ~/.ssh/config alias → windows.nuclearbomb6518.com:2222
```

- 원격 파일 읽기: `ssh windows "type C:\path\to\file"`
- 원격 명령 실행: `ssh windows "powershell -NoProfile -Command \"...\""` 또는 `ssh windows "cmd /c ..."`
- 파일 전송: `scp -P 2222 local_file <user>@windows.nuclearbomb6518.com:C:/dest/` (user → `secrets/credentials.md`)

### Cloudflare Tunnel (Windows)

- **Config 파일:** `C:\Users\<user>\.cloudflared\config.yml` (user → `secrets/credentials.md`)
- **Tunnel ID:** → `secrets/credentials.md`
- **도메인 추가 시:** config.yml에 ingress 항목 추가 → DNS에 CNAME 레코드 추가 (`{hostname}` → tunnel UUID.cfargotunnel.com) → `cloudflared` 서비스 재시작

```bash
# config.yml 읽기
ssh windows "type C:\Users\<user>\.cloudflared\config.yml"

# config.yml 수정 후 tunnel 재시작
ssh windows "net stop cloudflared && net start cloudflared"
```

현재 등록된 서비스 → 전체 목록은 `Infrastructure/services.md` 참조.

---

## MCP Servers

Configured in `.mcp.json`:

| Server | Purpose |
|--------|---------|
| `context7` | Latest library documentation lookup |
| `git` | Git history, blame, log access |
| `github` | PR/issue creation and retrieval via REST API |
| `memory` | Cross-session persistent memory |

Use `context7` when working with external libraries to get up-to-date docs.
Use `github` MCP for PR management (complementary to SSH git operations).

---

## Gstack Skills

Installed at `~/.claude/skills/gstack`. Use `/browse` for all web browsing — never use MCP browser tools directly.

| Skill | When to use |
|-------|------------|
| `/office-hours` | 새 아이디어/기능 초기 검토 (코딩 전) |
| `/plan-ceo-review` | 제품 관점 계획 검토 |
| `/plan-eng-review` | 스태프 엔지니어 관점 아키텍처 검토 |
| `/plan-design-review` | UI/UX 설계 검토 |
| `/qa` | 실제 브라우저 QA 자동화 실행 |
| `/cso` | OWASP + STRIDE 보안 감사 |
| `/ship` | PR 생성 + 배포 준비 |
| `/retro` | 작업 완료 후 회고 |
| `/benchmark` | 성능 벤치마크 |
| `/browse` | 웹 브라우징 (MCP 대신 사용) |
