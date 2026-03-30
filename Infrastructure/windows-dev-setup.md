# Windows AI Dev Environment Setup Guide

> Mac(`~/Documents/Work`) 환경과 동일한 AI 개발 환경을 Windows에 구성하는 가이드.

---

## 전체 구성 요소

| 컴포넌트 | Mac | Windows |
|---------|-----|---------|
| Shell | zsh (iTerm2) | PowerShell 7 + Windows Terminal |
| tmux | `brew install tmux` | WSL2 안에 설치 (권장) |
| Package manager | Homebrew | winget (기본 내장) |
| Node.js | `brew install node` | `winget install OpenJS.NodeJS` |
| Claude CLI | npm global | npm global (동일) |
| MCP 서버 | npx auto (`.mcp.json`) | npx auto (동일) |
| 설정 경로 `~/.claude/` | `/Users/js/.claude/` | `C:\Users\<이름>\.claude\` |
| 프로젝트 설정 | `Work/CLAUDE.md`, `.mcp.json` | git clone 후 그대로 사용 |

---

## Step 1 — Windows Terminal + PowerShell 7 설치

```powershell
# Windows Terminal (Microsoft Store 또는 winget)
winget install Microsoft.WindowsTerminal

# PowerShell 7 (최신 버전)
winget install Microsoft.PowerShell
```

이후 Windows Terminal을 기본 터미널로 설정하고, 기본 프로필을 PowerShell 7로 변경.

---

## Step 2 — WSL2 설치 (tmux 사용을 위해 필수)

tmux는 Windows 네이티브에서 동작하지 않으므로 WSL2(Ubuntu)를 사용.

```powershell
# PowerShell (관리자 권한)
wsl --install

# 재부팅 후 Ubuntu 실행 → username/password 설정
```

### WSL2 안에서 개발 도구 설치

```bash
# WSL2 Ubuntu 터미널 안에서
sudo apt update && sudo apt upgrade -y

# tmux
sudo apt install tmux -y

# zsh (선택)
sudo apt install zsh -y
chsh -s $(which zsh)

# Node.js (WSL2용)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install nodejs -y

# Git
sudo apt install git -y
```

> **팁**: Claude Code는 WSL2 안에서 실행하면 Mac과 거의 동일한 경험을 얻을 수 있음.

---

## Step 3 — 핵심 도구 설치 (Windows 네이티브 방식)

Claude Code는 Windows PowerShell에서도 직접 동작함.

```powershell
# Node.js 22 LTS
winget install OpenJS.NodeJS.LTS

# Git
winget install Git.Git

# GitHub CLI
winget install GitHub.cli

# Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Gemini CLI
npm install -g @google/gemini-cli

# OpenAI Codex CLI
npm install -g @openai/codex
```

### GitHub CLI 인증

```powershell
gh auth login
# GitHub.com → HTTPS → 브라우저로 인증
```

---

## Step 4 — 환경 변수 설정

### PowerShell 프로필에 추가

```powershell
# 프로필 파일 열기
notepad $PROFILE

# 아래 내용 추가 (토큰 값은 실제 값으로 교체)
$env:GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxxxxxxxxxxxxxxxxxxx"
```

또는 시스템 환경 변수로 영구 설정:

```powershell
# PowerShell (관리자 권한)
[System.Environment]::SetEnvironmentVariable(
  "GITHUB_PERSONAL_ACCESS_TOKEN",
  "ghp_xxxxxxxxxxxxxxxxxxxx",
  "User"
)
```

### WSL2 사용 시 `.bashrc` 또는 `.zshrc`에 추가

```bash
export GITHUB_PERSONAL_ACCESS_TOKEN="ghp_xxxxxxxxxxxxxxxxxxxx"
```

### GitHub PAT 권한 (github.com/settings/tokens에서 생성)

- `repo` — 전체 저장소 접근
- `read:org` — 조직 읽기
- `workflow` — GitHub Actions (선택)

---

## Step 5 — 프로젝트 파일 복사 (git clone)

프로젝트 설정 파일들(`CLAUDE.md`, `.mcp.json`, `.claude/`)은 git 저장소에 포함되어 있으므로 clone하면 자동으로 포함됨.

```powershell
# Windows PowerShell 또는 WSL2
git clone <repo-url> Work
cd Work
```

포함되는 파일:
```
Work/
├── CLAUDE.md                    ← Claude Code 전역 규칙
├── .mcp.json                    ← MCP 서버 설정 (자동 로드)
└── .claude/
    ├── agents/
    │   ├── file-explorer.md     ← /file-explorer sub-agent
    │   ├── architect.md         ← /architect sub-agent
    │   ├── code-reviewer.md     ← /code-reviewer sub-agent
    │   ├── frontend-developer.md
    │   ├── backend-developer.md
    │   ├── security-auditor.md
    │   ├── test-engineer.md
    │   └── devops-engineer.md
    └── commands/
        ├── commit.md            ← /commit
        ├── review.md            ← /review
        ├── plan.md              ← /plan
        ├── standup.md           ← /standup
        ├── explain.md           ← /explain
        ├── cleanup.md           ← /cleanup
        ├── e2e.md               ← /e2e
        ├── security-audit.md    ← /security-audit
        ├── ai-plan.md           ← /ai-plan
        └── ai-review.md         ← /ai-review
```

---

## Step 6 — 전역 Claude 설정 (`~/.claude/`)

Mac의 `~/.claude/`에 있는 전역 설정을 Windows에도 적용.

### Windows 경로: `C:\Users\<이름>\.claude\`

```powershell
mkdir $env:USERPROFILE\.claude
```

### 전역 메모리 폴더 생성

```powershell
mkdir $env:USERPROFILE\.claude\projects\-Users-js-Documents-Work\memory
```

> Mac의 memory 파일들을 복사하거나, 새 PC에서는 처음부터 쌓기 시작해도 됨.

---

## Step 7 — MCP 서버 확인

`.mcp.json`은 git에서 clone 후 자동으로 적용됨. 별도 설치 불필요 (npx가 자동으로 실행).

### 활성화된 MCP 서버

| 서버 | 패키지 | 용도 |
|------|--------|------|
| context7 | `@upstash/context7-mcp` | 라이브러리 최신 문서 |
| git | `@modelcontextprotocol/server-git` | git 히스토리/blame |
| github | `@modelcontextprotocol/server-github` | PR/이슈 REST API |
| memory | `@modelcontextprotocol/server-memory` | 세션 간 메모리 |
| pdf-reader | `@dev.saqibaziz/mcp-pdf-reader` | PDF 파일 읽기 |
| sequential-thinking | `@modelcontextprotocol/server-sequential-thinking` | 단계별 추론 |
| fetch | `@modelcontextprotocol/server-fetch` | URL 페이지 가져오기 |
| playwright | `@playwright/mcp` | 브라우저 자동화 |
| desktop-commander | `@wonderwhy-er/desktop-commander` | 터미널 제어 |

### MCP 서버 로드 확인

```powershell
# claude 세션 시작 후
claude mcp list
```

---

## Step 8 — tmux 세팅 (WSL2)

### tmux 기본 사용법 (Mac과 동일)

```bash
# WSL2 터미널에서
tmux new-session -s ai-workspace

# 패널 분할
Ctrl+B → %   # 좌우 분할
Ctrl+B → "   # 상하 분할
Ctrl+B → 방향키  # 패널 이동
Ctrl+B → d   # detach (세션 유지)
tmux attach -t ai-workspace  # 재연결
```

### `.tmux.conf` 설정 (선택)

```bash
# ~/.tmux.conf
set -g mouse on                    # 마우스 지원
set -g default-terminal "xterm-256color"
bind | split-window -h             # | 로 좌우 분할
bind - split-window -v             # - 로 상하 분할
```

### Windows Terminal 대안

tmux 없이 Windows Terminal의 자체 pane 분할 기능 사용 가능:
- `Alt+Shift+D` → 패널 분할
- `Alt+방향키` → 패널 이동

---

## Step 9 — Claude Code 실행

### PowerShell (Windows 네이티브)

```powershell
cd $env:USERPROFILE\Documents\Work
claude
```

### WSL2 (권장 — tmux 포함 Mac과 동일한 경험)

```bash
# WSL2 터미널
cd ~/Documents/Work   # WSL2 내 Work 폴더
claude
```

> WSL2에서 Windows 파일에 접근: `/mnt/c/Users/<이름>/Documents/Work`

---

## Step 10 — 검증 체크리스트

```powershell
# 1. Node.js 버전 확인
node --version   # 18+ 이어야 함

# 2. GitHub CLI 인증
gh auth status

# 3. Claude Code 버전
claude --version

# 4. Gemini CLI
gemini --version

# 5. git 설정
git config --global user.name "이름"
git config --global user.email "이메일"

# 6. MCP 서버 (claude 세션 안에서)
# claude mcp list

# 7. Slash commands 테스트 (claude 세션 안에서)
# /commit
# /review
# /standup
# /plan
```

---

## 트러블슈팅

### `claude` 명령이 PowerShell에서 안 될 때

```powershell
# npm global 경로를 PATH에 추가
$env:PATH += ";$(npm prefix -g)\bin"

# 또는 프로필에 영구 추가
echo '$env:PATH += ";$(npm prefix -g)\bin"' >> $PROFILE
```

### MCP 서버가 로드 안 될 때

- `GITHUB_PERSONAL_ACCESS_TOKEN` 환경 변수가 설정되어 있는지 확인
- `npx`가 정상 동작하는지 확인: `npx --version`

### WSL2에서 Windows 파일 경로 접근

```bash
# Windows의 C:\Users\이름\Documents\Work
cd /mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')/Documents/Work
```

### Git SSH 키 설정

```bash
# SSH 키 생성
ssh-keygen -t ed25519 -C "이메일@example.com"

# 공개키를 GitHub에 등록
cat ~/.ssh/id_ed25519.pub
# → github.com/settings/keys 에서 추가
```
