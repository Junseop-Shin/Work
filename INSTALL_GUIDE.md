# Install Guide — AI Dev Environment

> Required tools and setup steps for the `~/Documents/Work` AI development environment.

---

## Prerequisites

### 1. Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

---

## Core Tools

| Category | Tool | Install | Purpose |
|----------|------|---------|---------|
| **Runtime** | Node.js 18+ | `brew install node` | MCP server runtime |
| **GitHub CLI** | `gh` | `brew install gh` | PR creation and management |
| **AI: Claude** | `claude` | `npm install -g @anthropic-ai/claude-code` | Primary AI CLI |
| **AI: Antigravity** | `agy` | `brew install --cask antigravity-cli` | Antigravity CLI (Gemini CLI 후속) |
| **AI: Codex** | `codex` | `npm install -g @openai/codex` | OpenAI Codex CLI |

### Installation

```bash
# Runtime
brew install node

# GitHub CLI
brew install gh
gh auth login

# AI CLIs
npm install -g @anthropic-ai/claude-code
npm install -g @openai/codex
brew install --cask antigravity-cli
```

---

## MCP Servers

MCP servers are configured in `.mcp.json` and run automatically via `npx`. No global installation needed.

| Server | Package | Purpose | Auth Required |
|--------|---------|---------|---------------|
| **context7** | `@upstash/context7-mcp` | Latest library docs | No |
| **git** | `@modelcontextprotocol/server-git` | Git history, blame, log | No |
| **github** | `@modelcontextprotocol/server-github` | PR/issue REST API | Yes (PAT) |
| **memory** | `@modelcontextprotocol/server-memory` | Cross-session memory | No |
| **pdf-reader** | `@dev.saqibaziz/mcp-pdf-reader` | Local PDF text extraction | No |
| **sequential-thinking** | `@modelcontextprotocol/server-sequential-thinking` | Structured multi-step reasoning | No |
| **fetch** | `@modelcontextprotocol/server-fetch` | URL → clean text | No |
| **playwright** | `@playwright/mcp` | Browser automation | No |
| **desktop-commander** | `@wonderwhy-er/desktop-commander` | Terminal control, file search, diff edits | No |

### Verify MCP servers load

```bash
claude mcp list
```

---

## Environment Variables

### Required

```bash
# GitHub MCP server — PR/issue management via REST API
# SSH handles git protocol (push/pull); this is separate for the REST API
export GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```

Add to your shell profile (`~/.zshrc` or `~/.zprofile`):

```bash
echo 'export GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx' >> ~/.zshrc
source ~/.zshrc
```

### GitHub PAT Scopes Required

When creating the token at [github.com/settings/tokens](https://github.com/settings/tokens):
- `repo` — full repository access
- `read:org` — read org membership
- `workflow` — update GitHub Actions (optional)

---

## GitHub CLI Auth

```bash
gh auth login
# Choose: GitHub.com → HTTPS → Yes (authenticate with browser) → Login
```

Verify:
```bash
gh auth status
```

---

## Verification Checklist

After setup, verify each component:

```bash
# 1. Node.js version
node --version   # Should be 18+

# 2. GitHub CLI
gh auth status

# 3. Claude Code
claude --version

# 4. MCP servers (run inside claude session)
# Type: claude mcp list

# 5. Slash commands — inside claude session, try:
# /commit
# /review
# /standup
# /plan

# 6. Git history
git log --oneline
```

---

## Directory Structure Created

```
Work/
├── CLAUDE.md                        # Claude Code — canonical AI rules
├── AGENTS.md                        # Codex / Antigravity CLI config
├── README.md
├── INSTALL_GUIDE.md                 # This file
├── .mcp.json                        # MCP server configuration
├── .gitignore
├── setup_plan.md
└── .claude/
    ├── settings.json                # Shared settings (hooks, model)
    ├── settings.local.json          # Local permissions — gitignored
    ├── agents/
    │   ├── architect.md             # Opus — system design
    │   ├── backend-developer.md
    │   ├── code-reviewer.md         # Sonnet — code review
    │   ├── db-architect.md
    │   ├── devops-engineer.md
    │   ├── file-explorer.md         # Haiku — fast file search
    │   ├── frontend-developer.md
    │   ├── llm-engineer.md
    │   ├── playwright-engineer.md
    │   ├── security-auditor.md
    │   └── test-engineer.md
    └── commands/
        ├── plan.md                  # /plan
        ├── review.md                # /review
        ├── commit.md                # /commit
        ├── cleanup.md               # /cleanup
        ├── e2e.md                   # /e2e
        ├── explain.md               # /explain
        ├── security-audit.md        # /security-audit
        ├── standup.md               # /standup
        ├── journal.md               # /journal
        ├── work-history.md          # /work-history
        ├── study.md                 # /study
        ├── exam-study.md            # /exam-study
        └── notify-slack.md          # /notify-slack
```
