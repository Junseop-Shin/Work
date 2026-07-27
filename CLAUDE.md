# CLAUDE.md — AI Dev Environment Guidelines

> Configures Claude Code behavior for `~/Documents/Work`. Canonical source — other AI tool configs (AGENTS.md) summarize from here.
> Keep lean — review and trim periodically.

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

## Behavioral Principles

Four core behaviors derived from Andrej Karpathy's observations on LLM coding pitfalls. Bias toward caution over speed; for trivial tasks, use judgment.

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked
- No abstractions for single-use code
- No "flexibility" or "configurability" that wasn't requested
- No error handling for impossible scenarios
- If 200 lines could be 50, rewrite it

Test: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

- Don't "improve" adjacent code, comments, or formatting
- Don't refactor things that aren't broken
- Match existing style, even if you'd do it differently
- If you notice unrelated dead code, mention it — don't delete it
- Remove imports/variables/functions that YOUR changes made unused
- Don't remove pre-existing dead code unless asked

Test: Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

## Code Quality — additions to Behavioral Principles

- Trust framework guarantees — only validate at system boundaries
- Delete unused code completely — no backwards-compatibility shims
- No docstrings or comments for self-evident code

---

## Workflow Harnesses

Three optional systems are installed. Pick the one that fits the task — do not stack them. Trivial tasks (typo, obvious one-liner) do not need any harness.

| Harness | When to use |
|---------|------------|
| **Ouroboros** (`ooo interview`/`seed`/`run`/`evaluate`) | Spec-first work — turn a vague idea into a verified codebase via Socratic interview + 3-stage evaluation gate |
| **Superpowers** (`/brainstorm`, `/plan`, TDD/debugging skills) | Process discipline — brainstorm → plan → TDD → review |
| **Gstack** (`/qa`, `/ship`, `/cso`, `/browse`, `/retro`) | Operational — real-browser QA, deploy, security audit, web browsing |

Use `/browse` (Gstack) for all web browsing instead of MCP browser tools.

---

## Commit Rules

- Follow **Conventional Commits**: `type(scope): description`
  - Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`
- Never commit without explicit user approval
- Never force-push or `reset --hard` without explicit instruction
- Never commit to `main` directly — always use feature branches
- Branch naming: `feat/`, `fix/`, `docs/`, `chore/` prefix
- After commit: push branch + `gh pr create`. Merge is **manual**.

### Commit Message Format

```
type(scope): short description (max 72 chars)

- Bullet points for details if needed
- Reference issues: Closes #123

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## Language Policy

| Context | Language |
|---------|----------|
| Conversation with user | Korean OK (match user's language) |
| Code, variable names, code comments | English |
| Commit messages, PR titles/descriptions | `type(scope)` in English, description in Korean (e.g. `docs(study): 정처기 실기 정리 추가`) |
| Docs, directory names | Korean OK (code file names stay English) |

---

## Security Rules

- Never read or output contents of `.env`, `.env.*`, `*.pem`, `*.key`, `*.p12` files
- Never hardcode secrets, tokens, or credentials
- Use env var references (e.g., `$GITHUB_PERSONAL_ACCESS_TOKEN`) in configs
- Never disable security checks (`--no-verify`, skipping auth) without explicit instruction
- Validate user inputs at system boundaries; avoid OWASP Top 10 vulnerabilities

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

- Config: `C:\Users\<user>\.cloudflared\config.yml` (user → `secrets/credentials.md`)
- Tunnel ID → `secrets/credentials.md`
- 도메인 추가: config.yml에 ingress 추가 → DNS에 CNAME (`{hostname}` → `<tunnel-uuid>.cfargotunnel.com`) → 서비스 재시작

```bash
ssh windows "type C:\Users\<user>\.cloudflared\config.yml"
ssh windows "net stop cloudflared && net start cloudflared"
```

전체 서비스 목록은 `Infrastructure/services.md` 참조.

---

## MCP Servers

Configured in `.mcp.json`: `context7` (lib docs), `git` (history/blame), `github` (PR/issue), `memory` (cross-session). Use `context7` when working with external libraries.
