# Mac Workspace Setup Plan

> **AI Agent 자율 셋팅 플랜** — 단계별 진행 상황을 이 파일에 기록합니다.

---

## 전체 목표

Apple Silicon Mac의 개발 환경을 초기 셋팅하고, `~/Documents/Work`를
GitHub 블로그(TIL 방식) 저장소로 구축하고,
Claude Code를 주 AI 도구로 하는 멀티 AI 개발 환경을 구성한다.

---

## 접근 권한 정책

| 경로 | AI 접근 | 용도 |
|------|---------|------|
| `~/Documents/General/` | **금지** | 이력서, 인증서 원본, 금융 문서 등 민감 자료 |
| `~/Documents/Work/` | **허용** | AI 주 작업 공간 |

---

## 단계별 계획

### Phase 1 — 디렉토리 구조 생성 및 권한 분리 `[완료]`

- [x] `~/Documents/Work/` 및 하위 폴더 생성
  - `Study/` — SQLP, AZ-104 등 자격증·학습 자료
  - `Coding_Tests/` — 코딩 테스트 풀이
  - `Work_History/` — 경력 관련 문서
  - `Projects/` — 개인 프로젝트 (stock-automation, my-ui-lib 등)
  - `Infrastructure/` — IaC, 서버 설정, 아키텍처 문서
- [x] `General/`은 AI 미접근 영역으로 정책 명문화

**커밋 기준:** 디렉토리 skeleton + setup_plan.md 초안

---

### Phase 2 — GitHub 레포지토리 블로그화 `[완료]`

- [x] `~/Documents/Work`를 Git 저장소로 초기화
- [x] 대시보드 역할의 `README.md` 작성
  - 폴더별 하이퍼링크 목차
  - 자격증 인증 사진 첨부 가이드라인 포맷
- [x] `.gitignore` 생성 (OS 파일, Node modules, 민감 키 파일)
- [x] 초기 전체 커밋: `"Initial Mac Workspace Setup - Repo Blog Mode"`

---

### Phase 3 — AI 개발 환경 구축 `[완료]`

Claude Code를 주 AI 도구로 하는 멀티 AI 개발 환경 구성.

- [x] `CLAUDE.md` — 모델 선택 룰, 워크플로우, 행동 지침 (Canonical Source)
- [x] `.mcp.json` — context7, git, github, memory MCP 서버 설정
- [x] `.claude/commands/commit.md` — Conventional Commit 자동화
- [x] `.claude/commands/review.md` — 코드 리뷰 리포트 생성
- [x] `.claude/commands/plan.md` — Opus 기반 아키텍처 설계 문서
- [x] `.claude/commands/standup.md` — 일일 스탠드업 노트 생성
- [x] `.claude/agents/file-explorer.md` — Haiku 기반 파일탐색 서브에이전트
- [x] `.claude/agents/architect.md` — Opus 기반 설계 서브에이전트
- [x] `.claude/agents/code-reviewer.md` — Sonnet 기반 코드리뷰 서브에이전트
- [x] `GEMINI.md` — Gemini CLI 설정
- [x] `AGENTS.md` — OpenAI Codex CLI 설정
- [x] `.github/copilot-instructions.md` — GitHub Copilot 설정
- [x] `.cursor/rules/main.mdc` — Cursor/Antigravity IDE 룰
- [x] `.gitignore` 업데이트 — AI 도구 로컬 캐시 제외
- [x] `INSTALL_GUIDE.md` — 설치 필요 도구 목록

---

### Phase 4 — 향후 확장 (선택) `[예정]`

- [ ] GitHub Actions를 이용한 README 자동 갱신 (학습 일지 카운터 등)
- [ ] 각 서브폴더별 `README.md` 추가 (Projects, Study 등)
- [ ] Homebrew 기반 개발 도구 설치 스크립트 (`Brewfile`)
- [ ] dotfiles 관리 체계 구축
- [ ] GitHub Actions CI — lint + test 자동화

---

## 커밋 이력

| 커밋 | 내용 |
|------|------|
| `Initial Mac Workspace Setup - Repo Blog Mode` | Phase 1 + Phase 2 전체 초기 셋팅 완료 |
| `Add project repos to dashboard and link hub` | README 프로젝트 링크 추가 |
| `feat: Add CLAUDE.md` | 모델 룰, 워크플로우, 행동 지침 |
| `feat: Add .mcp.json` | context7, git, github, memory MCP 서버 |
| `feat: Add custom slash commands` | commit, review, plan, standup |
| `feat: Add custom sub-agents` | file-explorer, architect, code-reviewer |
| `feat: Add multi-AI config files` | GEMINI, AGENTS, Copilot, Cursor rules |
| `docs: Update setup_plan.md + .gitignore` | Phase 3 완료 기록 |
| `docs: Add INSTALL_GUIDE.md` | 설치 도구 목록 및 환경변수 설정 가이드 |

---

*마지막 업데이트: 2026-02-20*
