# GitHub Copilot Instructions

> Canonical rules are in `CLAUDE.md`. This file configures Copilot behavior for this workspace.

---

## Code Generation Rules

### Quality
- Write minimal code that solves the immediate problem
- No speculative features, future-proofing, or "nice to have" additions
- No unnecessary abstractions — inline simple operations
- Validate only at system boundaries (user input, external APIs)

### Security
- Never suggest hardcoded secrets, tokens, or credentials
- Use parameterized queries — never string-concatenate SQL
- Sanitize outputs to prevent XSS in web contexts
- Reference secrets via environment variables only

### Style
- Code and comments in English
- Follow existing naming conventions in the file
- Prefer explicit over implicit (clear variable names, no clever tricks)

---

## Commit & PR Standards

- Conventional Commits: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`
- Short, imperative commit titles (max 72 chars)
- Never include generated code without review

---

## Access Restrictions

- Do not suggest code that reads from `~/Documents/General/`
- Do not suggest reading or logging `.env` file contents

---

## Language

- Conversation and comments: Korean accepted
- Code, commits, PR descriptions: English only
