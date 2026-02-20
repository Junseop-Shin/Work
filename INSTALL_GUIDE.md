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
| **AI: Gemini** | `gemini` | `npm install -g @google/gemini-cli` | Gemini CLI |
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
npm install -g @google/gemini-cli
npm install -g @openai/codex
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
├── GEMINI.md                        # Gemini CLI config
├── AGENTS.md                        # OpenAI Codex CLI config
├── INSTALL_GUIDE.md                 # This file
├── .mcp.json                        # MCP server configuration
├── .gitignore
├── setup_plan.md
├── .github/
│   └── copilot-instructions.md      # GitHub Copilot config
├── .cursor/
│   └── rules/
│       └── main.mdc                 # Cursor/Antigravity rules
└── .claude/
    ├── agents/
    │   ├── file-explorer.md         # Haiku — fast file search
    │   ├── architect.md             # Opus — system design
    │   └── code-reviewer.md         # Sonnet — code review
    └── commands/
        ├── commit.md                # /commit
        ├── review.md                # /review
        ├── plan.md                  # /plan
        └── standup.md               # /standup
```
