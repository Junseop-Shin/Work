# GEMINI.md — Gemini CLI Configuration

> Canonical rules are in `CLAUDE.md`. This file summarizes key guidelines for Gemini CLI.

---

## Access Policy

- **Allowed**: `~/Documents/Work/` and all subdirectories
- **Forbidden**: `~/Documents/General/` — never read or reference

---

## Behavior Guidelines

### Code Quality
- Write minimal, focused solutions — no over-engineering
- No unnecessary abstractions, helpers, or error handling for impossible cases
- No docstrings or comments for self-evident code

### Security
- Never read, write, or output contents of `.env`, `*.pem`, `*.key` files
- Never hardcode secrets or tokens — use environment variable references
- Validate inputs only at system boundaries

### Commits & Changes
- Follow Conventional Commits: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`
- Never commit without explicit user approval
- Code and commit messages in English; conversation can be in Korean

---

## Workflow

1. Read relevant files before making changes
2. Plan before implementing (state the plan, get approval)
3. Make focused, minimal changes
4. Verify before committing

---

## Language Policy

| Context | Language |
|---------|----------|
| Conversation | Match user's language (Korean OK) |
| Code & commits | English only |

---

*For full guidelines, see [CLAUDE.md](./CLAUDE.md)*
