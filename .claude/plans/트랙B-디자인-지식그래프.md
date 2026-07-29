# 트랙 B — 디자인 지식그래프 + 조회·검증 하네스

## Context

참고 문서 4건이 같은 논지로 수렴한다: **모델을 바꾸는 대신 검증된 지식을 구조화해 에이전트가 조회하게 만들면 성능이 뛴다.**

| 출처 | 요지 |
|---|---|
| [Schema 하네스](https://www.aitimes.com/news/articleView.html?idxno=212889) | 모델 가중치 그대로, 실행 프레임워크만 바꿔 ARC-AGI-3 42.83% → 98.98% |
| [graphify](https://news.hada.io/topic?id=31743) | 데이터 → 지식그래프 → LLM 연결. AST 파싱, MCP 서버로 공유 |
| [brunch: 지어내지 말고 찾아 쓰라](https://brunch.co.kr/@kim-yezi/48) | 디자인 AI가 생성(generation) → 조회(retrieval)로. 디자인 시스템이 사람용 문서에서 **AI용 운영 입력값**으로 |
| [shadcn Base UI 전환](https://news.hada.io/topic?id=31163) | [트랙 C](./트랙C-base-ui-전환.md)의 근거 (별건) |

graphify는 "KG를 만들어주는 서비스"이고, 이 프로젝트는 그 서비스를 다시 짓는 게 아니라 **그 패턴으로 나올 산출물 하나** — UI/UX 도메인 지식을 그래프화해 실제 LLM에 붙여 쓰는 툴 — 을 만든다.

**목표**: 에이전트가 UI를 만들 때 규칙을 *조회*하고, 만든 결과를 같은 그래프로 *검증*하는 폐루프.

**성공 기준 (2단계 종료 시점)**: my-ui-lib 컴포넌트 1개를 대상으로, 그래프를 붙인 에이전트가 붙이지 않은 에이전트보다 **수치 규칙 위반 수가 적다.** 눈대중이 아니라 CHECKABLE 규칙 위반 카운트로 판정한다.

### 코퍼스

**[Madia Designer](https://www.youtube.com/@UXUIDesign)** — 한국어, 구독자 약 36.3만, 영상 약 1,120개.

- **포함**: 포트폴리오 리뷰, 웹 디자인, 모바일 디자인
- **제외**: Photoshop/Illustrator/XD 툴 조작법 — UI 코드를 만드는 에이전트에게 쓸모없음
- 리뷰 포맷이 핵심 소스다. before/after가 명시적이고 "왜 고쳤는지"를 말로 설명한다.

### 제약

1. **자막만으로는 추출 불가.** 강의 화법이 "여기를 이렇게 하면 낫죠?" 식 지시대명사 위주라 화면을 봐야 한다. 프레임 캡처가 필수고 이게 인제스천 단가를 결정한다.
2. **저작권.** 추출한 규칙(아이디어·사실)은 저작권 대상이 아니지만 자막 전문·원본 스크린샷 저장은 재배포다. Work 레포가 public이므로 **그래프에는 추출 결과 + 출처 링크(영상ID+타임스탬프)만 커밋**하고, 캡처 원본은 gitignore 대상 로컬 디렉터리에 둔다.

---

## 설계

### 왜 사례 → 규칙 승격인가

영상에서 "규칙 문장"을 직접 뽑으려 하면 안 나온다. 채널이 규칙을 명제로 말하지 않기 때문이다. 반면 "이 리뷰에서 뭘 지적했고 어떻게 고쳤나"는 잘 나온다.

그래서 **추출은 사례만, 규칙은 데이터가 만들게** 한다. 승격 조건은 두 가지 — 서로 다른 영상에서 **반복**되고, 화자가 **단호**했을 것. 이 두 축이 곧 오염 방지 게이트다. 한 영상의 개인 취향은 사례로 남고, "이렇게 해볼까요?" 수준은 선택지에 머물며, 반복되면서 단호했던 것만 규칙이 된다. `Projects/ontology-pipeline`의 SHACL이 하던 역할과 같다.

> ontology-pipeline README: *"사람이 만드는 것은 선언적 산출물 3개 + 정제 규칙. 스크립트는 범용 실행기. graph quality = 이 산출물 품질 = GraphRAG quality."*

이 순서를 그대로 따른다. **스키마 먼저, 인제스천 자동화는 나중.** 파이프라인을 먼저 만들면 스키마 확정 후 다시 짜게 되고, 영상 추출은 비싸서 재작업 비용이 크다.

### 노드

**`Case`** — 영상에서 추출한 단일 지적/개선 사례
```yaml
id: case-001
source: { video: <id>, t: "04:12" }
domain: mobile | web
component: button | card | list | nav | form | typo | color | ...
property: spacing | size | color | contrast | hierarchy | touch-target | ...
problem: 무엇이 문제였나
fix: 어떻게 고쳤나
rationale: 왜 (영상에서 말한 이유)
stance: PRESCRIPTIVE | PREFERRED | OPTION          # 화자 어조
quote: "터치 영역은 무조건 44 이상 잡으셔야 돼요"     # 어조 판정 근거 문장
measured: { prop: touch-target, value: 44, unit: px }   # 수치 언급 시에만
evidence: .evidence/<video>/0412.png                     # gitignore 대상
```

**`measured`** — "모바일에서 몇 픽셀, 여백은 얼마"처럼 수치가 언급되면 여기 담긴다. 서로 다른 영상의 수치가 일치·근접하면 그게 CHECKABLE 규칙이 된다.

**`stance`** — 화자 어조. 같은 내용도 어조에 따라 강제성이 다르다.

| stance | 화법 단서 (한국어 어미) | 의미 |
|---|---|---|
| `PRESCRIPTIVE` | "~해야 됩니다", "~하면 안 돼요", "무조건", "반드시", "절대" | **규칙 후보** |
| `PREFERRED` | "~하는 게 좋아요", "훨씬 낫죠", "추천드려요" | 선택지 (권장 표시) |
| `OPTION` | "~해볼까요?", "~해도 되고", "취향이에요", "이럴 수도 있고" | 선택지 |

**단호한 것만 규칙이 된다.** PREFERRED는 규칙으로 올리지 않고 Option 안에 `recommended` 표시로 남긴다. 규칙 수는 줄지만 검증이 거짓 위반을 뱉지 않는다 — 루프 신뢰도가 커버리지보다 우선이다.

어조는 자막 텍스트만으로 판별된다. 프레임 캡처가 필요한 다른 필드와 달리 추출 비용이 거의 없다.

**주의**: 어조는 화자의 확신도이지 규칙의 타당성이 아니다. 개인 취향일수록 더 단호하게 말하는 경향이 있다. 그래서 어조가 반복 게이트를 대체하지 않고 **두 축을 모두 통과해야** 규칙이 된다.

**`Rule`** — Case에서 승격된 규칙
```yaml
id: rule-007
statement: 모바일 터치 타겟은 최소 44px
grade: CHECKABLE | JUDGMENT
scope: { domain: mobile, component: button }
check: "min(width,height) >= 44"      # CHECKABLE만
promoted_from: [case-001, case-014, case-032]
confidence: 3                          # 근거 Case 수
```

**`Option`** — 단호하지 않은(OPTION·PREFERRED) Case가 반복될 때 승격되는 노드. 규칙이 아니라 **선택지 카탈로그**다.
```yaml
id: option-003
question: 카드 모서리 처리
choices:
  - { label: 라운드 8px, when: "친근·모바일", recommended: true, cases: [case-021, case-047] }
  - { label: 직각, when: "정보 밀도·데스크톱", cases: [case-055] }
scope: { component: card }
```
`recommended`는 PREFERRED 어조의 Case에서 온다 — "이쪽을 밀더라"는 정보를 버리지 않으면서 강제하지도 않는다.

조회 때는 "이런 선택지가 있고 각각 언제 쓴다"로 반환하고, **검증에서는 위반 판정을 하지 않는다.** 어느 쪽을 골라도 틀리지 않기 때문이다. 이게 없으면 취향 수준의 지적이 규칙으로 올라가 검증이 거짓 위반을 뱉는다.

**`Component`**, **`Property`** — Case/Rule/Option을 묶는 축. 조회 시 진입점 역할.

### 엣지

- `Case -[EXEMPLIFIES]-> Rule`, `Case -[ILLUSTRATES]-> Option`
- `Rule -[APPLIES_TO]-> Component`, `Rule -[ABOUT]-> Property`
- `Rule -[CONFLICTS_WITH]-> Rule` — 디자인 규칙은 서로 충돌한다(여백 확보 vs 정보 밀도). 충돌을 숨기지 말고 노드로 드러내 조회 시 함께 반환한다.
- `Case -[FROM]-> Video`

### 승격 게이트 — 2축

축 하나는 **반복**(서로 다른 영상 몇 개에서 나왔나), 다른 하나는 **어조**(화자가 얼마나 단호했나). 둘은 직교하므로 조합으로 판정한다.

| | OPTION / PREFERRED 우세 | PRESCRIPTIVE 우세 |
|---|---|---|
| **영상 3개 이상** | → `Option` 노드 | → `Rule` |
| **영상 1~2개** | Case로만 보관 | Case로만 보관 — 단호했지만 근거 부족 |

1. 서로 다른 **영상 3개 이상**의 Case가 같은 `(component, property, 방향)`을 지적 → 승격 후보
2. stance 다수결로 목적지 결정. **PRESCRIPTIVE가 우세할 때만 `Rule`**, 그 외는 전부 `Option`
3. Rule로 가는 후보 중 `measured` 있는 Case 2개 이상 + 값 일치/근접 → **CHECKABLE**, `check` 식 작성. 수치 없이 서술만 → **JUDGMENT**
4. **stance가 팽팽히 갈리면 `Option`으로 내린다.** 한 영상은 "무조건 이렇게", 다른 영상은 "해도 되고"라면 규칙이 아니다. `promotion-log.md`에 강등 사유를 남긴다
5. 내용이 서로 반대면(A는 하라, B는 하지 마라) `CONFLICTS_WITH`로 잇고 양쪽 다 조회에 노출한다
6. 승격 확정은 사람이 한다 (1단계는 전량 수동)

### 저장 형식 — 1단계는 파일, GraphDB 아님

20개 영상 × 사례 수 개 = 수백 노드 규모다. YAML 파일 + 로딩 스크립트로 충분하고, 이 규모에 GraphDB/RDF는 과잉이다. ontology-pipeline에서 GraphDB를 썼다고 여기서도 쓸 이유는 없다. 규모가 커지고 다중 홉 질의가 실제로 필요해질 때 옮긴다.

### 조회·검증 인터페이스 (MCP 툴 4개)

| 툴 | 입력 | 출력 |
|---|---|---|
| `design_rules` | component, platform, properties[] | 지켜야 할 규칙 목록. CHECKABLE 우선, CONFLICTS_WITH 동봉 |
| `design_options` | component, platform | 선택지 카탈로그 + `recommended` 표시. 강제 아님 |
| `design_cases` | rule_id \| option_id | 근거 사례 before/after 요약 |
| `design_check` | artifact (코드 또는 스펙) | **Rule만 판정.** CHECKABLE은 기계 판정, JUDGMENT는 근거 사례를 동봉해 LLM 판정. Option은 판정 대상 아님 |

조회는 규칙·선택지가, 검증은 거기 매달린 사례가 근거로 쓰인다. 판정 불가능한 "40%" 영역은 명제로 뭉개지 않고 **사례 비교**로 판정한다 — 말로는 불명확해도 결과물은 명확하기 때문.

조회와 검증의 대상 범위가 다른 게 핵심이다. **조회는 넓게(규칙+선택지), 검증은 좁게(규칙만).** 에이전트에게 줄 정보는 많을수록 좋지만, 위반으로 때릴 근거는 단호한 것만이어야 한다.

---

## 단계

### 1단계 — 스키마 확정 (수동 추출 20개)

영상 20개 선별(리뷰 중심 + 웹/모바일 디자인) → Case 추출 → 승격 시도 → 스키마 확정.

추출은 LLM 보조로 반자동(자막 + 프레임 캡처를 주고 Case 카드 초안 생성), 사람이 검수. **툴을 만드는 게 아니라 그때그때 돌린다.** 스키마 확정은 사람 몫.

산출물:
- `Projects/design-kg/schema.md` — 노드/엣지/승격 규칙 정의
- `Projects/design-kg/cases/*.yaml` — 추출된 Case (stance 포함)
- `Projects/design-kg/rules.yaml` — 승격된 Rule (PRESCRIPTIVE 우세만)
- `Projects/design-kg/options.yaml` — 승격된 Option
- `Projects/design-kg/promotion-log.md` — 무엇이 왜 승격/강등/보류됐나
- `.gitignore`에 `Projects/design-kg/.evidence/` 추가

**검증**: 승격 게이트를 20개 표본에 돌렸을 때 CHECKABLE 규칙이 최소 3개 나오는가.

규칙 문턱을 PRESCRIPTIVE로 좁혔으니 0개로 끝날 가능성이 실재한다. 그때는 원인을 구분해야 한다 — 코퍼스 선별이 틀렸으면 영상 선별 기준을 고치고, **채널이 원래 단호하게 말하지 않는 스타일이면 그건 결론이다.** 후자라면 이 코퍼스로는 검증 절반이 서지 않으므로, 검증용 규칙은 다른 출처(WCAG, 플랫폼 HIG 등 명문 규격)에서 가져오고 이 채널은 조회용 선택지 공급원으로 역할을 좁힌다. 2단계로 넘어가기 전에 이 판단을 먼저 내린다.

### 2단계 — LLM에 붙여 실효성 확인

최소 MCP 서버(툴 4개, YAML 읽어 응답)를 세우고 my-ui-lib 컴포넌트 1개로 조회 → 생성 → 검증 왕복 1회.

산출물: `Projects/design-kg/mcp/` + 왕복 기록

**검증**: 같은 컴포넌트를 (a) 그래프 없이 (b) 그래프 조회해서 각각 만들고, 두 결과를 `design_check`에 넣어 CHECKABLE 위반 수를 비교한다. 차이가 없으면 3단계로 가지 않는다 — 여기가 진짜 판정 지점이다.

### 3단계 — 인제스천 반자동화 (2단계 통과 시에만)

확정된 스키마로 영상 → Case 추출을 스크립트화. 자막 추출 + 프레임 캡처 + LLM 추출 + 사람 검수 큐.

### 4단계 — 지속 학습

승격 게이트를 자동 실행. 신규 Case 유입 → 후보 감지 → 사람 확정. 오염 없이 그래프가 자란다.

---

## 다른 트랙과의 관계

- **[트랙 C](./트랙C-base-ui-전환.md)** — 2단계 검증 대상이 my-ui-lib 컴포넌트이므로 C가 먼저 끝나 있는 게 깔끔하다. 강한 의존은 아니다.
- **[트랙 D](./트랙D-html-편집-익스텐션.md)** — 익스텐션이 `design_check`를 호출하는 그림이 가능하지만 **엮지 않는다.** 2단계 판정이 나온 뒤에 검토한다.
- **트랙 A (온톨로지 강의 정리)** — `Projects/ontology-pipeline` + `Work_History/2026-06-온톨로지-파이프라인-학습.md`로 이미 진행 중. 이 플랜은 거기서 배운 순서(스키마 → 매핑 → 게이트)를 빌려 쓴다.

## 미해결

- `share.google/eOSWKCWICXWtrv5FC` 링크가 구글 리다이렉트 오류로 열리지 않았다. 내용 확인 후 반영 필요.
- 영상 20개 구체 선별 목록은 1단계 착수 시 채널을 직접 훑어 정한다.
