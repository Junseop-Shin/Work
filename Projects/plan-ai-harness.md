# AI Dev Environment Harness — Setup Guide

> 이 문서를 Claude에게 보여주고 "이 문서대로 세팅해줘" 라고 하면
> 현재와 동일한 개발 환경 + AI 하네스를 자동으로 세팅합니다.
>
> Claude가 직접 실행하는 Runbook입니다.

---

## Claude가 이 문서를 실행할 때 따라야 할 순서

```
Step 0. OS 감지
Step 1. 작업 디렉토리 확인
Step 2. tmux/cmux 사용 여부 질문 (macOS/Linux만)
Step 3. 글로벌 Claude 설정 (settings.json + hooks)
Step 4. 에이전트 파일 생성
Step 5. 커맨드 파일 생성
Step 6. CLAUDE.md 생성
Step 7. 플러그인 설치 (Superpowers + Gstack)
Step 8. [tmux 선택 시] 터미널 환경 설정
Step 9. 검증
```

---

## Step 0. OS 감지

```bash
uname -s   # Darwin = macOS, Linux = Linux, MINGW/CYGWIN = Windows
```

- **Windows** → Step 8 (터미널 환경 설정) 전체 스킵. tmux/cmux 질문 생략.
- **macOS / Linux** → 모든 단계 진행.

---

## Step 1. 작업 디렉토리 확인

사용자에게 질문:

> "메인 작업 디렉토리가 어디인가요? (예: ~/Documents/Work)"

이후 모든 경로에서 `WORK_DIR` = 사용자가 답한 경로로 치환.
기본값: `~/Documents/Work`

```bash
mkdir -p "$WORK_DIR/.claude/agents"
mkdir -p "$WORK_DIR/.claude/commands"
mkdir -p ~/.claude/hooks
```

---

## Step 2. tmux/cmux 사용 여부 확인 (macOS/Linux만)

사용자에게 질문:

> "터미널 환경 설정 방식을 선택해주세요:
>
> A) **cmux + tmux 사용** — 멀티패널 터미널, AI 알림 연동, 세션 관리
> B) **VS Code / 기존 에디터만 사용** — 터미널 설정 없이 Claude Code만 세팅
>
> 어떤 걸 선택하시겠어요?"

- **A 선택** → Step 8에서 tmux/cmux 설정 진행
- **B 선택** → Step 8 스킵

---

## Step 3. 글로벌 Claude 설정

### 3-1. `~/.claude/settings.json`

**A (cmux 사용) 선택 시:**

```json
{
  "permissions": {
    "defaultMode": "auto"
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/workflow-check.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/cmux-notify.sh"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/cmux-notify.sh"
          }
        ]
      }
    ]
  }
}
```

**B (에디터만) 선택 시:**

```json
{
  "permissions": {
    "defaultMode": "auto"
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/workflow-check.sh"
          }
        ]
      }
    ]
  }
}
```

---

### 3-2. `~/.claude/hooks/workflow-check.sh`

**워크플로우 강제 게이트** — 계획 없이 코드 작성 방지

```bash
#!/usr/bin/env bash
# workflow-check.sh
# Claude Code PreToolUse hook — Plan → Test → Implement → Review 워크플로우 강제

INPUT=$(cat)

TOOL_NAME=$(python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    print(d.get('tool_name', ''))
except:
    print('')
" <<< "$INPUT" 2>/dev/null || echo "")

FILE_PATH=$(python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    print(d.get('tool_input', {}).get('file_path', ''))
except:
    print('')
" <<< "$INPUT" 2>/dev/null || echo "")

# 소스코드 파일만 체크
if [[ ! "$FILE_PATH" =~ \.(ts|tsx|js|jsx|py|go|java|kt|swift)$ ]]; then
  exit 0
fi

# 테스트 파일은 허용 (test-engineer가 작성하는 파일)
if [[ "$FILE_PATH" =~ \.(test|spec)\.(ts|tsx|js|jsx|py)$ ]] || \
   [[ "$FILE_PATH" =~ /__tests__/ ]] || \
   [[ "$FILE_PATH" =~ /e2e/ ]]; then
  exit 0
fi

FLAG="$HOME/.claude/workflow-approved"

if [[ ! -f "$FLAG" ]]; then
  if [[ "$TOOL_NAME" == "Write" ]]; then
    echo "🚫 WORKFLOW BLOCKED — 새 소스 파일 생성 전 플랜이 필요합니다."
    echo ""
    echo "필수 단계:"
    echo "  1. /plan    — 설계 (architect + 전문가 병렬 검토)"
    echo "  2. test-engineer 에이전트 — 테스트 먼저 작성"
    echo "  3. implement — 실제 코드 작성"
    echo "  4. /review  — 병렬 멀티에이전트 코드 리뷰"
    echo ""
    echo "플랜 완료 후 자동 승인되거나, 긴급 우회:"
    echo "  touch $FLAG"
    exit 2
  elif [[ "$TOOL_NAME" == "Edit" ]]; then
    echo "⚠️  WORKFLOW REMINDER: /plan → test-engineer → implement → /review"
    echo "플랜 완료 시: touch $FLAG  (이후 이 메시지 사라짐)"
  fi
fi

exit 0
```

```bash
chmod +x ~/.claude/hooks/workflow-check.sh
```

---

### 3-3. `~/.claude/hooks/cmux-notify.sh` (cmux 사용 시만)

```bash
#!/bin/bash
# cmux Claude Code hook — cmux 사이드바 알림 연동

CMUX_SOCK="$HOME/Library/Application Support/cmux/cmux.sock"
[ -S "$CMUX_SOCK" ] || exit 0

EVENT=$(cat)
EVENT_TYPE=$(echo "$EVENT" | jq -r '.hook_event_name // empty')

case "$EVENT_TYPE" in
  "Stop")
    cmux claude-hook stop 2>/dev/null || true
    ;;
  "Notification")
    cmux claude-hook notification 2>/dev/null || true
    ;;
esac
```

```bash
chmod +x ~/.claude/hooks/cmux-notify.sh
```

---

## Step 4. 에이전트 파일 생성

모두 `$WORK_DIR/.claude/agents/` 에 생성.

---

### `file-explorer.md`
```markdown
---
name: file-explorer
description: Fast file and code search specialist. Use for directory traversal, file pattern matching, content search, and quick lookups. Delegates to Haiku for cost efficiency. Invoke when you need to find files or search code without modifying anything.
model: claude-haiku-4-5-20251001
tools:
  - Glob
  - Grep
  - Read
  - Bash
---

# File Explorer Agent

You are a fast, efficient file and code search specialist. Your only job is to find things — files, functions, patterns, usages.

## Rules
- Use Glob for file pattern matching
- Use Grep for content search
- Use Read only to confirm a match, not to read full files
- Return results immediately — no unnecessary explanation
- Never modify files
```

---

### `architect.md`
```markdown
---
name: architect
description: Senior software architect for system design, architecture decisions, and technology selection. Uses Opus for deep reasoning on complex design problems. Invoke when you need architecture plans, technology trade-off analysis, or system design documents.
model: claude-opus-4-6
tools:
  - Glob
  - Grep
  - Read
  - WebSearch
  - WebFetch
---

# Architect Agent

You are a senior software architect. Produce clear, well-reasoned architecture documents.

## Approach
1. Understand constraints — performance, scalability, security, team size, timeline
2. Survey the landscape — research current best practices
3. Evaluate options — list alternatives with honest pros/cons
4. Make a recommendation — with clear rationale tied to constraints
5. Identify risks — and propose mitigations

## Design Principles
- Simplicity first — the best architecture is the simplest one that meets requirements
- Explicit over implicit — make dependencies and data flows visible
- Design for failure — assume components will fail
- Incremental delivery — prefer phased approaches

## Output Format

# Architecture Plan — <feature>
## Problem Statement
## Requirements
### Functional / Non-Functional
## Proposed Solution
## System Design
### Components | Data Flow | API Design
## Trade-offs Considered
## Implementation Plan
### Phase 1 — tasks with time estimates and verification methods
## Risk Assessment
## Open Questions
```

---

### `code-reviewer.md`
```markdown
---
name: code-reviewer
description: Code review and security audit specialist. Analyzes code for bugs, security vulnerabilities, code quality issues, and performance problems. Uses Sonnet for balanced depth and speed. Invoke when you need a thorough review of changed files or a security audit.
model: claude-sonnet-4-6
tools:
  - Glob
  - Grep
  - Read
  - Bash
---

# Code Reviewer Agent

You are a senior code reviewer. You find real bugs and issues, not style preferences.

## Focus Areas
- Logic errors and edge cases
- Null/undefined handling
- Performance anti-patterns (N+1, unnecessary re-renders, blocking I/O)
- Error handling completeness
- Code clarity and maintainability

## Output Format
For each issue: file path + line number + severity (Critical/Warning/Info) + explanation + fix suggestion.
```

---

### `security-auditor.md`
```markdown
---
name: security-auditor
description: Security specialist for vulnerability analysis, OWASP compliance, and risk assessment. Invoke when you need a security audit, before deploying sensitive features (auth, payments, file uploads), or after receiving a security report.
model: claude-sonnet-4-6
tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# Security Auditor Agent

You are a senior application security engineer. You audit code against OWASP Top 10 and CWE.

## Checklist
- Injection (SQL, shell, XSS)
- Broken Authentication (weak tokens, no expiry, no rate limiting)
- Sensitive Data Exposure (secrets in code, weak encryption)
- Broken Access Control (missing auth checks, IDOR)
- Security Misconfiguration (debug flags, permissive CORS)
- Vulnerable Dependencies (known CVEs)

## Output Format
Every finding: CWE ID + CVSS score + file:line + reproduction + fix.
```

---

### `test-engineer.md`
```markdown
---
name: test-engineer
description: Test generation specialist for unit, integration, and e2e tests. Invoke when you need to add test coverage, write tests for a new feature, or generate e2e test suites for UI flows.
model: claude-sonnet-4-6
tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash
---

# Test Engineer Agent

You write tests that catch real bugs, not tests that just pass.

## Rules
- Test behavior, not implementation
- Integration tests over unit tests for DB operations
- Feature-level test files: one file = one full feature flow
- TDD order: write failing test first, then implement
- All test comments in Korean

## Test Structure
describe('<기능>') → it('<시나리오>') → Arrange / Act / Assert

## Output
1. Test file with Korean comments
2. Setup (beforeEach, fixtures)
3. Happy path → edge cases → error cases
4. Run command
```

---

### `frontend-developer.md`
```markdown
---
name: frontend-developer
description: Frontend and UI specialist for React, TypeScript, React Native, and mobile development. Invoke when building UI components, handling state management, implementing responsive layouts, or working on mobile/web frontend features.
model: claude-sonnet-4-6
tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash
---

# Frontend Developer Agent

You are a senior frontend engineer specializing in React, TypeScript, React Native.

## Rules
- Always TypeScript — no `any` without explanation
- Functional components and hooks only
- Follow existing project patterns before introducing new ones
- Accessibility: ARIA labels, keyboard support
- Keep components small and single-responsibility

## Output Format
New component: TypeScript types + props interface + usage example comment.
Bug fix: root cause first → minimal fix → related issues to note.
```

---

### `backend-developer.md`
```markdown
---
name: backend-developer
description: Backend and API specialist for server-side development, database design, REST/GraphQL APIs, and system integration. Invoke when building API endpoints, database schemas, authentication flows, or backend business logic.
model: claude-sonnet-4-6
tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash
---

# Backend Developer Agent

You are a senior backend engineer specializing in API design, database modeling, and server-side architecture.

## Rules
- Validate all inputs at system boundaries — never trust client data
- Use parameterized queries — never string-interpolated SQL
- Handle errors explicitly — no swallowed exceptions
- Use transactions for multi-step DB operations
- Never hardcode secrets

## Output Format
New endpoints: route + request/response types + validation + handler + error cases.
DB changes: migration (up + down) + updated schema + affected queries.
```

---

### `devops-engineer.md`
```markdown
---
name: devops-engineer
description: DevOps and infrastructure specialist for CI/CD pipelines, Docker, Kubernetes, cloud deployment, and developer environment setup. Invoke when setting up deployment pipelines, debugging CI failures, writing Dockerfiles, or configuring cloud infrastructure.
model: claude-sonnet-4-6
tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash
---

# DevOps Engineer Agent

You build reliable, secure, and reproducible infrastructure.

## Rules
- Never hardcode secrets — always use secret references
- Prefer multi-stage Docker builds
- Always pin dependency versions in CI
- Fail fast in CI: lint → test → build → deploy
- Every change should be rollback-able

## Output Format
CI/CD: full YAML + required secrets + how to run locally.
Dockerfiles: multi-stage + non-root user + layer caching optimized.
Infrastructure: text diagram + cost implications + rollback procedure.
```

---

### `db-architect.md`
```markdown
---
name: db-architect
description: Database architecture specialist for schema design, data modeling, query optimization, and migration strategy. Invoke when designing new schemas, planning migrations, optimizing slow queries, or making decisions about database technology selection (PostgreSQL, MongoDB, Redis, TimescaleDB, Elasticsearch).
model: claude-opus-4-6
tools:
  - Glob
  - Grep
  - Read
  - WebSearch
  - WebFetch
---

# DB Architect Agent

You are a senior database architect specializing in relational, document, time-series, and search databases.

## Principles
- Schema follows access patterns, not the other way around
- Index strategically — every index has a write cost
- Normalize for integrity, denormalize for performance
- Additive migrations first, never destructive without rollback path

## Output Format
New schema: columns table + indexes table + relationships + access patterns + migration plan.
Migration: current state → target state → risk assessment → ordered steps + rollback → verification.
Query optimization: slow query + root cause + fix + expected improvement.
```

---

## Step 5. 커맨드 파일 생성

모두 `$WORK_DIR/.claude/commands/` 에 생성.

---

### `commit.md`
```markdown
# /commit — Conventional Commit Generator

Analyze staged changes and create a properly formatted commit, then push to a feature branch and open a PR.

## Steps

1. **Check staged changes**
   ```bash
   git diff --cached --stat
   git diff --cached
   ```

2. **Analyze changes** — determine commit type:
   - `feat` / `fix` / `docs` / `refactor` / `test` / `chore` / `ci`

3. **Draft commit message** (Conventional Commits):
   ```
   type(scope): short description (max 72 chars)

   - Detail bullet 1
   - Detail bullet 2

   Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
   ```

4. **Show the drafted message** — ask for approval before committing.

5. **After approval**, commit and push:
   ```bash
   git commit -m "<approved message>"
   git push -u origin <current-branch>
   ```

6. **Create PR** if on a feature branch (not `main`):
   ```bash
   gh pr create --title "<commit title>" --body "<summary>"
   ```

## Rules
- Never commit to `main` directly
- Never use `--no-verify`
- Always show message for approval before executing
- Set workflow-approved flag after commit: `touch ~/.claude/workflow-approved`
```

---

### `plan.md`
```markdown
# /plan — Architecture Design with Parallel Expert Review

Generate an architecture design document using the architect agent, then validate with parallel specialists.

## Steps

1. **Gather context** — read relevant existing files (Glob, Grep, Read)

2. **Generate initial design** — delegate to `architect` agent (Opus) via Task tool

3. **Parallel specialist review** — once architect completes, launch in parallel:
   - `security-auditor` — security implications, threat surface
   - `backend-developer` — API design, DB schema, scalability
   - `devops-engineer` — deployment complexity, infra requirements
   - `frontend-developer` — if UI components involved

4. **Synthesize and present**:
   - Architect's design document
   - Specialist Feedback section with concerns
   - Blockers that require redesign

5. **Set workflow-approved flag** once user approves:
   ```bash
   touch ~/.claude/workflow-approved
   ```

## Rules
- Always run architect + security-auditor + backend-developer + devops-engineer
- Implementation tasks must be bite-sized: single responsibility + time estimate + verification
- Do not start implementation until user explicitly approves
```

---

### `review.md`
```markdown
# /review — Parallel Multi-Agent Code Review

Analyze changed files using parallel specialist agents.

## Steps

1. **Identify changed files**
   ```bash
   git diff --name-only HEAD
   ```

2. **Launch specialist agents in parallel** via Task tool:

   Always run:
   - `code-reviewer` — bugs, logic errors, performance
   - `security-auditor` — OWASP Top 10, CWE IDs
   - `test-engineer` — coverage gaps

   Conditionally:
   - `frontend-developer` — if `.tsx/.jsx/.css` detected
   - `backend-developer` — if server-side `.ts/.py/.go/.java` detected

3. **Aggregate results** — merge findings, deduplicate, tag source agent.

4. **Output unified report**:
   ```
   ## Code Review Report — <date>
   Files: X | Agents: Y | Issues: Z (Critical: A, Warning: B, Info: C)

   ### 🔴 Critical
   - [SEC] file.ts:42 — description
     > security-auditor · CWE-89

   ### 🟡 Warnings
   ### 🔵 Info
   ### ✅ Looks Good
   ```

5. **Do not auto-fix** — report only.

6. **Reset workflow flag**:
   ```bash
   rm -f ~/.claude/workflow-approved
   ```
```

---

### `security-audit.md`
```markdown
# /security-audit — OWASP Security Audit

## Steps

1. Identify target files (argument or `git diff --name-only HEAD`)
2. Read each file
3. Run dependency audit: `npm audit --audit-level=high 2>/dev/null || true`
4. Check for hardcoded secrets
5. Audit against OWASP Top 10:
   - Injection, Broken Auth, Sensitive Data, Broken Access Control
   - Security Misconfiguration, XSS, Vulnerable Dependencies, Missing Headers

6. Generate report:
   ```
   ## Security Audit — <date>
   Files: X | Findings: Y (Critical/High/Medium/Low)
   ### Critical / Dependencies / Approved
   ```

## Rules
- Report only — do not auto-fix
- Every finding: file + line + severity + remediation
```

---

### `standup.md`
```markdown
# /standup — Daily Standup Note Generator

## Steps

1. Get today's commits:
   ```bash
   git log --since="midnight" --pretty=format:"%h %s"
   ```

2. If none, check yesterday: `git log --since="yesterday" -10`

3. Generate report:
   ```markdown
   ## Daily Standup — <date>
   ### Yesterday
   - <completed work>
   ### Today
   - <planned work>
   ### Blockers
   - <blockers or "None">
   ```
```

---

### `cleanup.md`
```markdown
# /cleanup — Dead Code and Import Cleanup

## Steps

1. Target: argument provided, or `git diff --name-only HEAD`
2. Read each file fully
3. Remove:
   - Unused imports
   - Dead functions/variables (never called)
   - Commented-out code
   - `console.log` debug statements
   - TypeScript: unnecessary type annotations, `as any` casts
4. Report what was removed

## Rules
- Only remove **provably unused** code — no guessing
- Do NOT refactor logic — only remove dead code
- Run `tsc --noEmit` after if available
```

---

### `notify-slack.md`
```markdown
# /notify-slack — Slack 알림 전송

작업 완료, 승인 요청, 빌드 결과 등을 Slack으로 전송한다.

## Steps

1. **메시지 구성** — `$ARGUMENTS` 가 있으면 그대로 사용. 없으면 현재 컨텍스트에서 자동 생성:
   - 직전에 완료한 작업 요약 (1-3줄)
   - 승인이 필요하면 "✅ 승인 필요: [무엇을]" 형태로
   - 빌드/배포 결과면 성공/실패 + 링크 포함

2. **Webhook URL 로드**
   ```bash
   SLACK_WEBHOOK_URL=$(grep "SLACK_WEBHOOK_URL" ~/Documents/Work/Projects/devops-monitor/.env | cut -d'=' -f2-)
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
```

---

## Step 6. `CLAUDE.md` 생성

`$WORK_DIR/CLAUDE.md` 에 생성. 프로젝트마다 별도 CLAUDE.md를 추가로 둘 수 있음.

```markdown
# CLAUDE.md — AI Dev Environment Guidelines

## Access Policy

| Path | AI Access |
|------|-----------|
| `~/Documents/General/` | FORBIDDEN |
| `$WORK_DIR/` | ALLOWED |

- Never read `.env` files or files containing secrets
- Never modify CI/CD pipeline configurations without explicit approval

---

## Model Selection Rules

| Task Type | Model |
|-----------|-------|
| File search, quick lookups | Haiku |
| Code writing, debugging | Sonnet |
| Architecture design, complex analysis | Opus |
| Sub-agent: file-explorer | Haiku |
| Sub-agent: architect, db-architect | Opus |
| Sub-agent: others | Sonnet |

---

## Workflow (MUST follow — do not skip)

1. **Understand** — Read relevant files before changes
2. **Plan** — `/plan` for non-trivial tasks (triggers architect + specialists)
3. **Test First** — `test-engineer` agent writes tests before implementation
4. **Implement** — make focused, minimal changes
5. **Review** — `/review` (parallel agents)
6. **Commit** — only after explicit user approval via `/commit`

`workflow-check.sh` hook enforces this — blocks new file creation without plan approval.
Approve a session: `touch ~/.claude/workflow-approved`
Reset after review: `rm -f ~/.claude/workflow-approved`

---

## Commit Rules

- Conventional Commits: `type(scope): description`
- Never commit to `main` — always use feature branches (`feat/`, `fix/`, `docs/`)
- Never force-push without explicit instruction
- PR title follows same format

---

## Language Policy

| Context | Language |
|---------|----------|
| Conversation | Korean OK |
| Code, comments | English |
| Commit messages | English |
| PR titles/descriptions | English |

---

## Sub-Agent Usage

| Agent | When to invoke |
|-------|---------------|
| `file-explorer` | File search, grep |
| `architect` | System design decisions |
| `db-architect` | Schema design, migrations, query optimization |
| `code-reviewer` | Code quality review |
| `security-auditor` | Security audit |
| `test-engineer` | Writing tests |
| `frontend-developer` | React/RN UI work |
| `backend-developer` | API/DB/server work |
| `devops-engineer` | CI/CD, Docker, infra |

---

## Custom Commands

| Command | Purpose |
|---------|---------|
| `/plan` | Architecture design + parallel expert review |
| `/review` | Parallel multi-agent code review |
| `/commit` | Conventional Commit + push + PR |
| `/security-audit` | OWASP audit |
| `/standup` | Daily standup notes |
| `/cleanup` | Dead code removal |
| `/notify-slack` | 작업 완료/승인 요청/빌드 결과를 Slack으로 전송 |

---

## Security Rules

- Never output `.env`, `.pem`, `.key` file contents
- Never hardcode secrets, tokens, or credentials
- Validate all user inputs at system boundaries
- Avoid SQL injection, XSS, command injection
```

---

## Step 7. 플러그인 설치

### Superpowers (설계 강제 게이트)

```bash
claude /plugin marketplace add obra/superpowers-marketplace
claude /plugin install superpowers@superpowers-marketplace
```

설치 확인:
```bash
cat ~/.claude/plugins/installed_plugins.json | python3 -m json.tool
```

주요 커맨드: `/brainstorm` — 새 기능 요구사항 정제 (승인 전 코딩 불가)

### Gstack (브라우저 QA + 회고 + 벤치마크)

```bash
git clone https://github.com/garrytan/gstack ~/.claude/skills/gstack
```

CLAUDE.md 의 MCP 섹션 또는 settings.json에 등록 필요 (gstack 문서 참조).

주요 커맨드:
- `/browse` — 헤드리스 브라우저 (MCP 대신 사용)
- `/qa` — 브라우저 QA 자동화
- `/retro` — 스프린트 회고
- `/office-hours` — 제품 관점 아이디어 검토

---

## Step 8. 터미널 환경 설정 (tmux/cmux 선택 시만)

> Windows → 이 단계 전체 스킵

### 8-1. tmux 설치 확인

```bash
which tmux || brew install tmux   # macOS
# or: sudo apt install tmux       # Ubuntu/Debian
```

### 8-2. cmux 설치

cmux 공식 문서/다운로드 페이지에서 설치 (macOS 전용 앱).
설치 후 실행 → 사이드바에서 Claude Code 알림 연동됨.

### 8-3. `~/.zshrc` 에 추가

```bash
# ─── tmux auto-start ──────────────────────────────────────────────────────────
if command -v tmux &>/dev/null && [[ -z "$TMUX" ]] && [[ "$TERM_PROGRAM" != "vscode" ]]; then
  if [[ -n "$CMUX_WORKSPACE_ID" ]]; then
    # cmux에서 열린 새 탭 → 새 tmux 세션 자동 생성
    tmux new-session -s "work-$(date +%s)" -c ~/Documents/Work
  else
    # 일반 터미널 → main 세션 attach (없으면 생성)
    tmux attach-session -t main 2>/dev/null || tmux new-session -s main -c ~/Documents/Work
  fi
fi

# ─── tmux shortcuts ───────────────────────────────────────────────────────────
alias ta='tmux attach-session -t'
alias tn='tmux new-session -s'
alias tl='tmux list-sessions'
alias tk='tmux kill-session -t'
alias td='tmux detach'
alias tks='tmux kill-server'
```

```bash
source ~/.zshrc
```

---

## Step 9. 검증

세팅 완료 후 아래 항목을 순서대로 확인:

```bash
# 1. 에이전트 파일 확인
ls $WORK_DIR/.claude/agents/
# 예상: architect.md, backend-developer.md, code-reviewer.md, db-architect.md,
#        devops-engineer.md, file-explorer.md, frontend-developer.md,
#        security-auditor.md, test-engineer.md

# 2. 커맨드 파일 확인
ls $WORK_DIR/.claude/commands/
# 예상: cleanup.md, commit.md, plan.md,
#        review.md, security-audit.md, standup.md

# 3. 훅 파일 확인
ls ~/.claude/hooks/
# 예상: workflow-check.sh (+ cmux-notify.sh if cmux 선택)

# 4. settings.json 확인
cat ~/.claude/settings.json | python3 -m json.tool

# 5. CLAUDE.md 확인
cat $WORK_DIR/CLAUDE.md | head -20

# 6. 워크플로우 훅 동작 테스트
# Claude에게: "test.ts 파일 하나 만들어줘"
# → 🚫 WORKFLOW BLOCKED 메시지가 나와야 정상
# → touch ~/.claude/workflow-approved 후 재시도하면 통과

# 7. Superpowers 확인
cat ~/.claude/plugins/installed_plugins.json | python3 -m json.tool
```

---

## 전체 개발 워크플로우 (완성 후)

```
새 기능 시작
  │
  ▼
/office-hours (Gstack)          ← 제품 관점 아이디어 검토
  │
  ▼
/plan                           ← architect 초안 + 전문가 병렬 검토 → 승인 시 workflow-approved 플래그 생성
  │
  ▼
test-engineer 에이전트           ← 테스트 먼저 작성 (TDD RED)
  │
  ▼
구현 (Claude Code)              ← TDD GREEN → REFACTOR
  │
  ▼
/review                         ← 병렬 멀티에이전트 코드 리뷰 → workflow-approved 플래그 초기화
  │
  ▼
/commit                         ← Conventional Commit + push + PR
  │
  ▼
/retro (Gstack)                 ← 작업 회고
```

---

## 참고: 병렬 개발 패턴 (Conductor)

독립 기능을 동시에 개발할 때:

```bash
# 피처별 Git Worktree 격리
git worktree add ../project-auth feature/auth
git worktree add ../project-dashboard feature/dashboard

# 터미널 탭 분리
Tab 1: Claude Code → feature/auth 디렉토리
Tab 2: Claude Code → feature/dashboard 디렉토리
Tab 3: Antigravity CLI (agy) → 전체 아키텍처 분석 (대용량 컨텍스트)
```
