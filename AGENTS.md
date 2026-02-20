# AGENTS.md — OpenAI Codex CLI Configuration

> Canonical rules are in `CLAUDE.md`. This file summarizes key guidelines for OpenAI Codex CLI (`codex` command).

---

## Access Policy

- **Allowed**: `~/Documents/Work/` and all subdirectories
- **Forbidden**: `~/Documents/General/` — do not access

---

## Agent Behavior

### Before Making Changes
1. Read the relevant files first
2. State the planned approach and get approval for non-trivial changes
3. Make minimal, focused edits

### Code Standards
- Solve the problem at hand — no speculative features
- No unnecessary abstractions for one-time use
- Validate inputs at system boundaries only; trust internal code
- No backwards-compatibility shims for removed code

### Security
- Never expose or output secret values from `.env`, `*.pem`, `*.key`
- Use environment variable references in configs (`$VAR_NAME`)
- Guard against OWASP Top 10 vulnerabilities in any code you write

---

## Commit Policy

- Conventional Commits format: `type(scope): description`
- Require explicit user approval before committing
- Never push to `main` directly — use feature branches
- English for all code, commit messages, and PR content

---

## Language

- Conversation: match user language (Korean accepted)
- Code, commits, PRs: English

---

*For full guidelines, see [CLAUDE.md](./CLAUDE.md)*
