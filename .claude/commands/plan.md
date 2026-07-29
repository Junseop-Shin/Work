# /plan — Architecture Design with Parallel Expert Review

Generate an architecture design document using the architect agent, then validate it with parallel specialist agents before presenting to the user.

## Steps

1. **Gather context** — read relevant existing files:
   - Project structure (`Glob`, `Grep`)
   - Existing architecture patterns
   - Related code files
   - Any existing design documents

2. **Generate initial design** — delegate to `architect` agent (Opus) via Task tool:
   - Full problem description
   - Constraints and requirements
   - Current codebase context
   - Produce full architecture document (see format below)

3. **Parallel specialist review** — once architect completes, launch in parallel via Task tool:
   - `security-auditor` — review design for security implications, threat surface, missing auth/authz
   - `backend-developer` — review for backend feasibility, API design, DB schema, scalability
   - `devops-engineer` — review for deployment complexity, infra requirements, CI/CD impact
   - `frontend-developer` — if UI components involved, review for frontend feasibility and UX concerns

   Each specialist receives: the architect's full design document + original requirements

4. **Synthesize and present**:
   - Present the architect's design document
   - Append a **"Specialist Feedback"** section with concerns from each agent
   - Highlight any conflicts or blockers (items that would require redesign)

5. **Wait for user approval** before any implementation begins

6. **워크플로우 승인 플래그 생성** — 유저가 플랜을 승인하면 즉시 실행:

```bash
touch ~/.claude/workflow-approved
```

그리고 알려줘: "✅ 플랜 승인됨 — 이제 코드 작성 가능합니다. (test-engineer → implement → /review 순서로 진행)"

> 이 플래그는 `/review` 완료 시 삭제됩니다. `~/.claude/hooks/workflow-check.sh`가 이 플래그로 소스 파일 생성을 게이트합니다.

## Architecture Document Format

```markdown
# Architecture Plan — <feature/task name>
## Date: <date>

## Problem Statement
What we're solving and why.

## Requirements
### Functional
- ...
### Non-Functional
- Performance, security, scalability constraints

## Proposed Solution
High-level approach and rationale.

## System Design
### Components
| Component | Responsibility | Technology |
|-----------|---------------|------------|

### Data Flow
1. Step 1
2. Step 2

### API/Interface Design
Key interfaces or APIs if applicable.

## Trade-offs Considered
| Option | Pros | Cons | Decision |
|--------|------|------|---------|

## Implementation Plan
### Phase 1 — <name>
- [ ] Task (Xmin) — verification: ...
- [ ] Task (Xmin) — verification: ...

### Phase 2 — <name>
- [ ] Task (Xmin) — verification: ...

## Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|

## Open Questions
- Question 1
```

## Specialist Feedback Format

```markdown
## Specialist Feedback

### Security (security-auditor)
- ⚠️ Concern: ...
- ✅ Looks good: ...

### Backend (backend-developer)
- ⚠️ Concern: ...

### DevOps (devops-engineer)
- ⚠️ Concern: ...

### Frontend (frontend-developer) — if applicable
- ⚠️ Concern: ...

### Blockers (must resolve before implementation)
- [ ] ...
```

## Rules

- Always run architect + security-auditor + backend-developer + devops-engineer
- Add frontend-developer only if the design involves UI components
- Implementation tasks must be bite-sized: single responsibility + time estimate + verification method
- Do not start implementation until user explicitly approves
- Flag open questions rather than making assumptions
