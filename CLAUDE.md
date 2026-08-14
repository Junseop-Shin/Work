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
- Merge method: **squash** (`gh pr merge --squash --delete-branch`) — one commit per PR on `main`.
  Split commits inside the PR for review; they collapse on merge.

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

### 한글 문서 문체

`Study/`·`Work_History/` 등 사용자가 읽을 문서에 적용한다. 내용이 맞아도 문체에서 기계가 쓴 티가
나면 신뢰가 깎인다. 초안을 쓸 때부터 적용한다 — 다 쓰고 나서 고치면 절 구조까지 흔들린다.

- **볼드 남발.** 문단마다 두세 군데씩 굵게 하지 않는다. 한 문단에 많아야 하나
- **`✅ 🔶 ❌` 같은 판정 기호.** 표 안에서도 쓰지 말고 "있다 / 없다 / 절반만"처럼 말로 쓴다
- **은유의 반복.** 눈금·축·자리·결(結)·층 같은 비유어를 문서 전체에서 돌려쓰면 그 자체가 AI
  냄새가 된다. 기준·지표·지점·부분처럼 평이한 말로
- **한자어 상투어** — 정산하다·회수하다·병기하다·판정하다·곧장·짚어두다·받쳐주다·덮다(cover)·
  무너뜨리다 → 확인하다·둘 다 적어두다·정하다·바로·적어두다·뒷받침하다·차지하다·뒤집다
- **문장 끝 패턴 반복** — "~라는 뜻이다 / ~인 셈이다 / ~라는 것이다 / 여기서 ~가 나온다".
  대신 평서형으로 끊는다
- **대시(—)로 삽입구 붙이기.** 문장을 끊어서 따로 쓴다
- 절마다 요약 한 줄로 닫는 습관. 문장 길이가 다 비슷한 것도 티가 난다

어투 수정은 산문에만 적용한다. 표와 mermaid 다이어그램은 그대로 두고, 특히 자료(논문 Table 등)를
옮긴 표는 원형을 지킨다.

### 문서 내 도형·표기

계층·흐름·그래프·아키텍처는 실제 도형으로 그린다. 표로 옮기면 계층과 흐름이 죽는다.

- 기본은 ` ```mermaid ` 펜스. mermaid로 안 되는 그림은 인라인 SVG
- 표로 대체하지 않는다. 표는 항목 나열용이다. ASCII 아트도 쓰지 않는다
- 코드블록은 진짜 코드일 때만 언어 태그를 붙인다 (turtle, SPARQL 등)

**HTML 태그 함정** — 표 셀 안의 `<br>`은 사용자 뷰어에 그대로 노출된다. `·`로 잇는다.
mermaid 노드 라벨의 `<b>`·`<i>`도 그대로 렌더된다. 라벨은 순수 텍스트로 쓰고, 강조가 필요하면
단어 자체를 길게 쓴다 (`EL` → `OWL 2 EL`). `<br/>`는 mermaid 라벨에서만 정상 동작한다 —
표 셀과 다르니 헷갈리지 말 것.

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

Configured in `.mcp.json`: `context7` (lib docs), `git` (history/blame), `github` (PR/issue), `memory` (cross-session), `pdf-reader`, `sequential-thinking`, `fetch`, `playwright`, `desktop-commander`. Use `context7` when working with external libraries.
