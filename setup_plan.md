# Mac Workspace Setup Plan

> **AI Agent 자율 셋팅 플랜** — 단계별 진행 상황을 이 파일에 기록합니다.

---

## 전체 목표

Apple Silicon Mac의 개발 환경을 초기 셋팅하고, `~/Documents/Work`를
GitHub 블로그(TIL 방식) 저장소로 구축한다.

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

### Phase 3 — 향후 확장 (선택) `[예정]`

- [ ] GitHub Actions를 이용한 README 자동 갱신 (학습 일지 카운터 등)
- [ ] 각 서브폴더별 `README.md` 추가 (Projects, Study 등)
- [ ] Homebrew 기반 개발 도구 설치 스크립트 (`Brewfile`)
- [ ] dotfiles 관리 체계 구축

---

## 커밋 이력

| 커밋 | 내용 |
|------|------|
| `Initial Mac Workspace Setup - Repo Blog Mode` | Phase 1 + Phase 2 전체 초기 셋팅 완료 |

---

*마지막 업데이트: 2026-02-20*
