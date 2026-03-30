# MCP Ecosystem — Design Document

> AI 에이전트 전용 도구 세트. 로컬 리소스(코드, 기획, 디자인)에 직접 접근하여 판단하고 실행.
> 작성일: 2026-03-28 | 리뷰 반영: 2026-03-28

---

## 환경 & 인프라

| 항목 | 내용 |
|------|------|
| OS | Windows (i5-14400, 32GB RAM, Intel UHD 730 128MB) |
| LLM | Ollama CPU 모드 → `qwen2.5-coder:7b` (Q4_K_M, ~4.5GB RAM) |
| MCP 실행 | Windows 네이티브 (Node.js + Playwright) — Mac 테스트 동일 방식 |
| Ollama | Windows native → `localhost:11434` (MCP와 동일 머신) |
| Claude 연결 | Claude Desktop ↔ MCP (stdio) |
| 패키지 | pnpm monorepo + TypeScript + tsx |
| 모델 교체 | env 변수(`OLLAMA_MODEL`)로 관리 → 언제든 교체 가능 |

### Ollama 설정
```bash
# Windows 환경변수
OLLAMA_HOST=127.0.0.1:11434   # 로컬만 허용 (네트워크 노출 차단)
```

### Config 구조 (공개 레포 대응)

레포에 포함 (예시값만):
```
config/
  services.example.json    ← 서비스 URL, Confluence/Stitch 설정 예시
  auth.example.json        ← 로그인 정보 구조 예시
```

레포 밖 또는 gitignore (실제값):
```
~/.config/mcp-ecosystem/
  services.json            ← 실제 서비스 설정
  auth.json                ← 실제 로그인 정보
```

**services.json**
```json
{
  "targets": [
    {
      "name": "my-service",
      "baseUrl": "https://dev.mycompany.com",
      "srcDir": "C:/projects/my-service/src",
      "authRef": "my-service"
    }
  ],
  "confluence": {
    "baseUrl": "https://mycompany.atlassian.net",
    "spaceKey": "MYSPACE"
  },
  "stitch": {
    "projectId": "my-stitch-project"
  },
  "ollama": {
    "baseUrl": "http://localhost:11434",
    "model": "qwen2.5-coder:7b"
  }
}
```

**auth.json** (gitignore)
```json
{
  "services": {
    "my-service": {
      "username": "test@company.com",
      "password": "testpassword"
    }
  },
  "confluence": { "apiToken": "..." },
  "stitch": { "apiKey": "..." }
}
```

**Claude Desktop env** (API 토큰 주입)
```json
{
  "mcpServers": {
    "analyst": {
      "command": "pnpm",
      "args": ["--filter", "analyst-mcp", "start"],
      "cwd": "C:/projects/mcp-ecosystem",
      "env": {
        "MCP_CONFIG_PATH": "C:/Users/me/.config/mcp-ecosystem"
      }
    }
  }
}
```

---

## 아키텍처 원칙

- Tools는 **데이터 수집/정제만**, PRD·설계 생성은 Claude가 담당
- Ollama는 raw 데이터 → 구조화 JSON 압축 레이어
  - **컨텍스트 효율:** 7b 모델 컨텍스트 윈도우(8k~16k) 내에 맞추기
  - **Claude API 비용 절감:** 정제된 JSON만 Claude에 전달
- 분석 결과는 **Knowledge Base**에 라우트 단위로 누적 저장
- 최종 출력: 문서 → **Confluence**, 디자인 → **Google Stitch**

---

## ⚠️ 미결 사항 (구현 전 결정 필요)

### [해결됨] Figma → Google Stitch 교체
Figma REST API 쓰기 불가 문제 → **Google Stitch**로 대체

- TypeScript SDK (`stitch-sdk`) — 프로그래밍 방식 UI 생성
- MCP 서버 지원 (`stitch-mcp`) — AI 에이전트 직접 연동
- 텍스트 프롬프트 → UI 화면 자동 생성
- HTML + 스크린샷 내보내기 가능

```
Phase 3 design_spec (텍스트) → Stitch SDK → UI 화면 생성 → HTML 내보내기
```

### [블로커] 오케스트레이션 레이어
5개 MCP 서버가 독립 동작 → 단계 간 자동 연결 없음.

→ **구현 계획: `pipeline.ts` 스크립트 (monorepo root)** — 단계 순서 실행 + Knowledge Base 상태 체크 + 진행 로그

---

## 전체 데이터 흐름

```
서비스 화면 + 코드
      │
      ▼
[Phase 1: Analyst]    → PRD (라우트별)          → Confluence
      │
      ▼
[Phase 2: Librarian]  → 디자인 시스템             → Confluence (Figma 대체)
      │
      ▼
[Phase 3: Visualizer] → 디자인 문서 (라우트별)    → Confluence (Figma 대체)
      │
      ▼
[TDD Gate]            → 기획 기반 테스트 먼저 작성 → 구현 전 테스트 통과 확인
      │
      ▼
[구현]                → 개발 진행
      │
      ▼
[Phase 4: Architect]  → 정합성 검토 + 실행계획    → Confluence
      │
      ▼
[Phase 5: Healer]     → 에러 분석 + 수정 제안     → Confluence
```

---

## Knowledge Base 구조

```
knowledge-base/
  manifest.json                  ← 라우트별 메타데이터 (타임스탬프, Confluence ID, 완료 상태)
  prds/                          ← 기획 (라우트별 캐시)
    login.json
    dashboard.json
    user-detail.json
  design/
    _system/                     ← 전역 디자인 시스템
      tokens.json
      theme.json
      components/
    pages/                       ← 디자인 문서 (라우트별 캐시)
      login.json
      dashboard.json
      user-detail.json
```

> Knowledge Base = 외부 서비스 장애 대비 로컬 캐시 + 중간 처리 레이어

### manifest.json 구조
```json
{
  "/login": {
    "lastAnalyzedAt": "2026-03-28T00:00:00Z",
    "phaseStatus": { "analyst": "done", "librarian": "pending" },
    "confluencePageId": "12345",
    "schemaVersion": "1.0"
  }
}
```

### Knowledge Base JSON 스키마
**구현 시작 전 `packages/shared/src/schemas/` 에 TypeScript 타입 먼저 정의**
- `PrdSchema` — Phase 1 출력 / Phase 4 입력 계약
- `DesignSystemSchema` — Phase 2 출력
- `DesignPageSchema` — Phase 3 출력

---

## Confluence 구조

```
Space: 서비스명
  ├─ 기획 (PRD)
  │    ├─ /login
  │    ├─ /dashboard
  │    └─ /user/:id
  ├─ 디자인 시스템
  └─ 실행계획 / 리포트
```

API: Confluence Cloud REST API v2
- 페이지 생성: `POST /wiki/api/v2/pages`
- 재분석 시 업데이트: `PUT /wiki/api/v2/pages/{id}` (ID는 manifest.json에서 조회)

---

## Phase 1: Analyst MCP

**목표:** 서비스 화면 + 코드 분석 → PRD JSON 역생성 → Confluence

### Tools

| Tool | 입력 | 출력 |
|------|------|------|
| `list_routes` | src_dir | 전체 라우트 목록 |
| `mcp_login` | service_name | 브라우저 헤드 모드로 수동 로그인 → storageState 저장 |
| `collect_ui` | url | A11y Tree JSON (JSP는 DOM fallback 자동 적용) |
| `collect_interactions` | url, scenarios? | 클릭→API 매핑 (핵심) |
| `collect_source` | keyword, src_dir | Tree-sitter AST 기반 코드 스니펫 (보조) |
| `refine` | raw 3개 결과 | 압축된 기능 명세 JSON |
| `store_prd` | route, prd | Knowledge Base 저장 |
| `publish_confluence` | route, prd | Confluence 페이지 생성/업데이트 |

### collect_source — Tree-sitter 기반 AST 파싱 + 보안 제한

> **Gemini 제안 반영:** grep 패턴 대신 **Tree-sitter** 사용 — 언어 무관 AST 파서
> regex로 못 잡는 중첩 구문, 다중 데코레이터, 복잡한 타입 시그니처도 정확히 추출

| 언어 | Tree-sitter 문법 | 추출 노드 |
|------|----------------|---------|
| TypeScript/TSX | `tree-sitter-typescript` | event_handler, call_expression(fetch/axios), export_statement |
| JavaScript/JSX | `tree-sitter-javascript` | event_handler, call_expression, export_statement |
| Java | `tree-sitter-java` | annotation(@RequestMapping 등), method_declaration |
| Python | `tree-sitter-python` | decorator, function_definition |
| JSP | regex fallback | form action, $.ajax, fetch( |

```typescript
// collect_source 구현 방향
import Parser from 'tree-sitter'
import TypeScript from 'tree-sitter-typescript'

const parser = new Parser()
parser.setLanguage(TypeScript.typescript)
const tree = parser.parse(sourceCode)
// AST 순회 → 이벤트 핸들러 / API 호출 노드 추출
```

**보안 제한 (구현 필수):**
- 파일 확장자 allowlist만 읽기 (위 목록 외 거부)
- denylist: `*.env*`, `*.key`, `*.pem`, `*secret*`, `*credential*`, `*token*`
- `src_dir` path traversal 방지 (사전 승인된 경로만 허용)
- LLM 전달 전 주석 제거 (프롬프트 인젝션 방지)

### collect_interactions 주의사항
- **테스트 환경 필수** — 실서비스 연결 시 삭제/결제 버튼 부작용 위험
- **화이트리스트** — role=button/link 중 destructive 키워드(삭제, 결제, 전송) 필터링
- **WebSocket** — `page.on('websocket', ...)` 별도 등록
- **인증** — `storageState`로 로그인 상태 유지 (최초 인증은 `mcp_login` 도구 사용)
- **React 대기** — `waitUntil: 'networkidle'` 표준화 (JSP는 `domcontentloaded`)

### mcp_login — 최초 인증 도구

> **Gemini 제안 반영:** auth.json 패스워드 직접 사용 대신, 최초 1회 헤드 브라우저로 수동 로그인

```
mcp_login("my-service")
  → chromium headed 모드 실행 (headless: false)
  → 사용자가 직접 로그인 (2FA, SSO 포함)
  → 완료 후 storageState를 ~/.config/mcp-ecosystem/sessions/my-service.json 저장
  → 이후 collect_ui / collect_interactions는 storageState만 사용 (패스워드 불필요)
```

- **2FA/SSO 서비스에 필수** (auth.json 패스워드로는 불가)
- storageState 만료 시 재실행만 하면 됨
- auth.json에서 `password` 필드는 storageState 없는 headless 서비스용으로만 유지

### 우선순위
- **1순위:** `collect_interactions` (런타임 캡처 — 스파게티 코드 관통)
- **2순위:** `collect_ui`
- **3순위:** `collect_source` (역추적 보조, 파일당 최대 20줄 / 3파일 제한)

---

## TDD Gate (Phase 3 → 구현 사이)

**목표:** 신규 기획/디자인 문서 생성 후 구현 전, 기획 명세 기반 테스트 먼저 작성

```
Phase 3 완료 (PRD + 디자인 문서 생성)
  │
  ▼
🔴 RED — PRD 기반 테스트 작성 (test-engineer 에이전트 활용)
  │       - 기능 명세의 각 조건 → 테스트 케이스
  │       - API 호출 검증 (collect_interactions 결과 활용)
  │       - 에러 케이스 커버
  ▼
구현 시작
  │
  ▼
🟢 GREEN — 테스트 통과하는 최소 구현
  │
  ▼
🔵 REFACTOR — 품질 개선 (테스트 유지)
  │
  ▼
Phase 4: Architect 정합성 검토
```

---

## Phase 2: Librarian MCP

**목표:** 코드베이스 → 디자인 시스템 추출 → Confluence

### Tools

| Tool | 입력 | 출력 |
|------|------|------|
| `extract_tokens` | src_dir | colors, fonts, spacing 등 |
| `map_component_usage` | token, src_dir | 토큰 사용 컴포넌트 목록 (최대 50개 제한) |
| `refine_design_system` | raw_tokens | primitive/semantic/component 3계층 |
| `publish_confluence` | design_system | Confluence 디자인 시스템 페이지 |

---

## Phase 3: Visualizer MCP

**목표:** 디자인 일관성 검토 + Google Stitch로 UI 생성 → Stitch 프로젝트 (라우트별)

> **사용 빈도: LOW** — 주로 최초 디자인 생성 시 또는 Phase 4에서 신규 기능 설계 시에만 사용.
> 기존 서비스 분석(Phase 1~2) 루틴에는 포함 안 됨. MVP 구현 우선순위 낮음.

### Tools

| Tool | 입력 | 출력 |
|------|------|------|
| `check_design_fit` | new_design_spec | 불일치 항목 + tone_score |
| `generate_design_spec` | new_prd, route | 일관된 새 디자인 명세 |
| `validate_consistency` | prd, tokens | PRD ↔ 디자인 정합성 |
| `publish_stitch` | route, design_spec | Stitch SDK → AI UI 화면 생성 + HTML 내보내기 |

> **디자인 도메인 내부** 일관성만 담당
> Google Stitch TypeScript SDK로 텍스트 명세 → UI 화면 자동 생성

---

## Phase 4: Architect MCP (컨트롤 타워)

**목표:** 기획-디자인-코드 크로스 도메인 정합성 + 실행계획 → Confluence

### Tools

| Tool | 입력 | 출력 |
|------|------|------|
| `check_planning_consistency` | new_prd | 기존 PRD 충돌/중복 여부 |
| `detect_domain_conflicts` | prd, design, code (3쌍 분리 호출) | 3방향 충돌 목록 |
| `generate_execution_plan` | conflicts | 우선순위 + 실행계획 |
| `publish_confluence` | plan | Confluence 실행계획 페이지 |

> **크로스 도메인** (기획 vs 디자인 vs 코드) 정합성 담당
> `detect_domain_conflicts` — 7b 한계로 3쌍 분리 호출 후 Claude가 병합

---

## Phase 5: Healer MCP

**목표:** 런타임 에러 → PRD 대조 → 자가 치유 → Confluence

### Tools

| Tool | 입력 | 출력 |
|------|------|------|
| `analyze_error` | error_log, src_dir | 에러 파싱 + 관련 파일 |
| `compare_with_prd` | error_context | 명세 이탈 리포트 |
| `suggest_fix` | deviation | 수정 제안 + 코드 패치 |
| `run_validation` | url, scenario | Gstack `/qa`로 브라우저 QA 실행 |
| `publish_confluence` | report | Confluence 수정 리포트 |

### 자가 치유 루프
```
에러 감지 → analyze_error → compare_with_prd
→ suggest_fix → 개발자 승인
→ run_validation (Gstack /qa)
→ 통과: 종료 / 실패: 재시도 (최대 3회, exponential backoff)
→ 3회 초과: 중단 + Confluence 실패 리포트
```

---

## Shared 패키지

```
packages/shared/src/
  ollama.ts        Ollama 호출 공통
                   - OLLAMA_BASE_URL env (기본: localhost:11434)
                   - 입력 3000 토큰 상한 (초과 시 interactions→UI→source 순 truncation)
                   - JSON fence 추출 + Zod 검증 + 1회 재시도
                   - 90초 timeout + health check (서버 시작 시 확인)
  config.ts        config/ 파일 로더 (services.json + auth.json)
                   - MCP_CONFIG_PATH env로 경로 지정
                   - 없으면 ~/.config/mcp-ecosystem/ 기본값
  playwright.ts    Playwright 표준 launch 옵션 중앙화
                   - headless: true (일반 수집) / false (mcp_login 최초 인증)
                   - storageState 우선 사용, 없을 때 auth.json 패스워드로 headless 로그인
                   - mcp_login: headed 브라우저 실행 → 수동 로그인 후 storageState 저장
  mcp.ts           MCP 서버 초기화 래퍼
  confluence.ts    Confluence REST API (exponential backoff 포함)
  stitch.ts        Google Stitch SDK 래퍼 (design_spec → UI 생성 → HTML 내보내기)
  schemas/         TypeScript 타입 (구현 시작 전 먼저 정의)
    prd.ts
    design-system.ts
    design-page.ts
    config.ts        ← services.json / auth.json 스키마
```

---

## 프로젝트 구조

```
mcp-ecosystem/
  pnpm-workspace.yaml
  package.json
  pipeline.ts              ← 오케스트레이션 스크립트 (단계 순서 실행)
  scripts/
    setup.sh               ← 초기 설정 (playwright install chromium 등)
  knowledge-base/
    manifest.json
    prds/
    design/
  packages/
    shared/
    analyst-mcp/
    librarian-mcp/
    visualizer-mcp/
    architect-mcp/
    healer-mcp/
```

---

## MVP 구현 순서

```
0. schemas/ 타입 정의 먼저
1. shared/ 패키지 (ollama.ts, playwright.ts, path.ts)
2. Analyst MCP — 단일 라우트 end-to-end 검증
   - mcp_login → storageState 확보
   - collect_ui + collect_interactions + collect_source (Tree-sitter) + refine
   - Confluence publish
3. TDD Gate — 테스트 작성 후 구현 루틴 정착
4. pipeline.ts 기본 오케스트레이션
5. Librarian MCP → Architect MCP → Healer MCP 순차 구현
6. Visualizer MCP — 신규 기능 설계 필요 시 (저빈도)
```
