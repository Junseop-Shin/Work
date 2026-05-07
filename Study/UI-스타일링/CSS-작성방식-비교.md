# CSS 작성 방식 비교 (Tailwind / CSS Modules / CSS-in-JS / CVA)

> 2026-05-08 | TailwindCSS, CSSModules, CSSinJS, CVA, ContainerQuery, JIT

## 한 줄 요약

2020년대 들어 **컴포넌트가 의미 단위가 되면서** semantic CSS의 5가지 함정(이름 짓기·co-location·dead CSS·specificity·시스템 강제)이 드러났다. **Tailwind**의 utility-first가 이걸 동시에 해결하며 표준이 됨. **CSS-in-JS(styled-components/Emotion)는 runtime cost + RSC 충돌**로 사형선고를 받아 zero-runtime 후예(Vanilla Extract, Linaria, Panda)로 정신만 살아남았다. **CSS Modules**는 specificity·충돌만 해결하는 무난한 차선. **2026 디폴트 조합 = Tailwind + CVA + tailwind-merge + clsx**.

## 핵심 개념

### 왜 semantic CSS가 무너졌나 — 컴포넌트 시대의 도래

90년대 CSS 정설은 "구조(HTML) + 스타일(CSS) 분리". 한 페이지가 의미 단위였고, CSS 클래스가 그 의미를 표현:

```css
.header { ... }
.user-card { ... }
.product-list { ... }
```

깔끔했음. 그런데 2010년대 후반 React/Vue 컴포넌트 시대가 오면서 **의미 표현이 컴포넌트로 이동**:

```tsx
<Header />
<UserCard />
<ProductList />
```

이제 CSS 클래스는 **의미를 표현할 이유가 없음**. 의미는 컴포넌트가 이미 가짐. 그런데 사람들은 관성으로 semantic을 시도했고, **5가지 함정**이 드러났음.

#### Semantic CSS의 5가지 함정

| 함정 | 증상 |
|---|---|
| **① 이름 짓기 지옥** | `.user-card-info-name-wrapper` ··· BEM/OOCSS/SMACSS 명명 규약이 줄줄이 시도됐지만 인지 비용 폭발 |
| **② 스타일이 컴포넌트와 멀어짐** | `Button.tsx` ↔ `Button.module.css` 분리 — 점프 비용 |
| **③ Dead CSS 누적** | "이 클래스 어디서 쓰는지 모름 → 안전하게 안 지움" → CSS 파일이 시간 지날수록 비대 |
| **④ Specificity 전쟁** | `.btn` vs `.modal .btn` vs `.modal-special .btn !important` → cascade 깨지면 더 깨짐 |
| **⑤ 디자인 시스템과 결합 ↓** | `padding: 17px` 누가 박으면 시스템(8/16/24)에서 벗어남, 강제할 도구 없음 |

#### Utility-first가 5개를 동시에 해결

| 함정 | utility 해법 |
|---|---|
| ① 이름 짓기 | `flex p-4 rounded-lg` — 이름 안 지음 |
| ② 멀어짐 | 컴포넌트 파일에 inline → **co-location** |
| ③ Dead CSS | utility는 정해진 집합 + JIT가 안 쓰는 거 자동 제거 |
| ④ Specificity | 모두 `.class` 단일 specificity → 평등 |
| ⑤ 시스템 강제 | `p-4`만 가능 (`p-17` 없음) → **시스템 강제됨** |

**핵심 통찰:** 컴포넌트 시대에 CSS 클래스의 "의미" 역할이 사라지면서, 의미 표현 비용을 안 낼 수 있게 된 것. 이게 utility-first의 본질.

---

### "긴 className은 더러움" 비판의 답

```html
<div class="flex items-center gap-4 p-6 rounded-lg bg-white shadow-md hover:shadow-lg dark:bg-gray-800 md:p-8">
```

**Tailwind 진영의 정당화 3가지:**

**① 추상화는 컴포넌트가 한다, 클래스가 아니다**

```tsx
// 정의 (1번)
function Card({ children }) {
  return <div className="flex items-center gap-4 p-6 rounded-lg bg-white shadow-md ...">{children}</div>
}

// 사용 (수십 번)
<Card>...</Card>
<Card>...</Card>
```

semantic CSS의 "DRY"를 utility는 **컴포넌트로 해결**. CSS 레벨 추상화 X, 컴포넌트 레벨 추상화 O. **이중 추상화 (CSS class + 컴포넌트) 사라짐.**

**② Co-location 이득**

"이 element가 어떻게 보이는지" → 다른 파일 안 봄. 5초 만에 파악.

**③ 도구 압축**

- `prettier-plugin-tailwindcss` — 클래스 자동 정렬
- VS Code IntelliSense — hover 시 실제 CSS 표시
- `clsx` / `tailwind-merge` / CVA — 조건부 + 충돌 + variant 표준화

긴 문자열의 가독성 비용 < co-location + 시스템 강제 + 인지 비용 절감.

---

### JIT 컴파일러 — File Size 문제 해결

**문제:** utility는 모든 가능한 조합을 미리 정의 가능. spacing × color × breakpoint × hover/focus/dark 등 → 카르테시안 곱이 수십만 개 → 순진하게 다 출력하면 CSS 파일이 메가바이트.

**Tailwind v1/v2 (2019-2020) — PurgeCSS 후처리**

```
[1단계] 빌드 시 모든 utility 생성 → CSS 파일 3MB
[2단계] PurgeCSS가 소스 파일 정규식 스캔 → 사용된 토큰만 추출
[3단계] 추출 토큰의 CSS만 남김 → 최종 10KB
```

작동은 하는데 빌드마다 3MB 생성 + 제거 → 느림.

**Tailwind v3 (2021+) — JIT (Just-In-Time)**

```
[파일 변경 감지]
   → 소스 스캔 (className 토큰 추출)
   → 매칭 utility만 생성
   → 핫 리로드
```

빌드 시간 0.1초, dev 즉시. 부수 효과로 **arbitrary values** 가능:

```html
<div class="w-[762px] bg-[#1e293b] rotate-[7deg]">
```

`w-762`는 미리 정의 안 되어있어도 JIT가 즉석 생성. 디자이너 시안의 임의값 표현 가능 — 단, 명시적 대괄호라 "system에서 벗어남"이 코드 리뷰에서 보임.

**핵심:** "**utility 카르테시안 곱은 무한하지만 실제 쓰는 건 유한**".

---

### Tailwind가 다른 utility 라이브러리를 이긴 이유

Utility-first 자체는 새 발상이 아님 — Atomic CSS / OOCSS (Yahoo, 2009), **Basscss** (2013), **Tachyons** (2014) 가 선배. Tailwind 빼고 다 죽었음.

**① 디자인 시스템 커스터마이징 (격차의 핵심)**

Tachyons는 고정 스케일 — 색 30개, spacing 8단계 하드코딩. 회사 디자인에 맞추려면 fork해서 빌드해야 함. 사실상 못 씀.

```js
// tailwind.config.js — 모든 걸 노출
export default {
  theme: {
    colors: { brand: { 50: "#eff6ff", 500: "#3b82f6" } },
    spacing: { 1: "4px", 2: "8px", 3: "12px", 4: "16px" },
  },
}
```

→ 회사 토큰을 그대로 utility로. **시스템 강제 + 커스터마이징 모두 가능.**

**② PostCSS/JIT 컴파일러 — "스타일 컴파일러" 패러다임**

선배: "고정 CSS 파일을 import" (정적). Tailwind: "내 코드에 맞춰 CSS 생성" (동적). → JIT, arbitrary values, custom variants의 진화 가능.

**③ Variant 시스템**

```html
<button class="bg-blue-500 hover:bg-blue-600 focus:ring-2 dark:bg-blue-700 md:bg-blue-400 group-hover:scale-110 peer-checked:bg-green-500 data-[state=open]:bg-red-500">
```

`hover:`, `focus:`, `disabled:`, `dark:`, `md:`, `group-`, `peer-`, `data-[...]` ··· 조합 폭발. 표현력 격차 압도적.

**④ Timing (컴포넌트 시대 도래)**

- Tachyons (2014) — React 막 떠오를 무렵
- Tailwind (2017) — Hooks 1년 전, 컴포넌트가 상식

utility-first의 정당성은 "**컴포넌트가 의미를 갖는다**"에 있는데, 그 전제가 강했던 시기에 등장.

**⑤ Adam Wathan의 advocacy**

창립자가 에세이 작가. *"CSS Utility Classes and Separation of Concerns"* (2017), 책 *Refactoring UI* 까지 묶어 "왜"를 설득. 단순 라이브러리 vs 패러다임을 설득한 라이브러리의 차이.

**한 줄:** Tachyons는 **CSS 라이브러리**, Tailwind는 **CSS 컴파일러 + 디자인 시스템 도구 + 생태계**.

---

### 반응형 — Mobile-First + Breakpoint Variant

**Tailwind 디폴트 = mobile-first.** prefix 없으면 baseline (모바일), prefix 붙으면 **min-width** 이상에서 적용.

```html
<div class="
  text-sm           ← 기본 (모바일)
  md:text-base      ← 768px 이상
  lg:text-lg        ← 1024px 이상
">
```

```css
.text-sm { font-size: 0.875rem; }
@media (min-width: 768px) { .md\:text-base { font-size: 1rem; } }
@media (min-width: 1024px) { .lg\:text-lg { font-size: 1.125rem; } }
```

**왜 mobile-first?**
- 작은 화면이 콘텐츠 우선순위 명확 → 큰 화면은 "더해지는" 디자인
- max-width 모델은 desktop이 baseline, mobile이 override → 모바일 깨지기 쉬움
- mobile-first는 **쌓아올리는 모델**, 점진적 enhancement

**기본 breakpoint (수정 가능):**
| prefix | min-width | 의미 |
|---|---|---|
| (none) | 0 | 모바일 |
| `sm:` | 640px | 큰 모바일 |
| `md:` | 768px | 태블릿 |
| `lg:` | 1024px | 데스크톱 |
| `xl:` | 1280px | 큰 데스크톱 |
| `2xl:` | 1536px | 매우 큼 |

---

### Container Query — 부모 크기 기준 반응형

**미디어쿼리의 한계:** viewport(브라우저 창) 크기 기준. 같은 viewport라도 부모 컨테이너 크기는 다름.

```
[사이드바 안의 카드] vs [메인 콘텐츠 안의 같은 카드]
   같은 viewport, 다른 부모 크기 → 다른 레이아웃이어야 함
```

**Container Query** (CSS 신기능, 2023+ 모든 모던 브라우저). Tailwind v3.2+ 의 `@tailwindcss/container-queries` 플러그인 (v4 내장):

```html
<div class="@container">                ← 컨테이너로 지정
  <div class="
    flex flex-col              ← 좁을 때: 세로
    @md:flex-row               ← 컨테이너가 768px 이상이면 가로
  ">
    ...
  </div>
</div>
```

| | 미디어쿼리 (`md:`) | 컨테이너쿼리 (`@md:`) |
|---|---|---|
| 기준 | viewport | 가장 가까운 `@container` 부모 |
| 사용 시점 | 페이지 레이아웃 | 컴포넌트 내부 |
| 컴포넌트 재사용 | 위치마다 다른 코드 | 한 컴포넌트로 어디서든 적응 |

**일반 추천:** 페이지 레이아웃은 미디어쿼리, 컴포넌트 내부는 컨테이너쿼리. 경쟁 X, 보완 O.

---

### CSS-in-JS의 흥망

styled-components / Emotion이 2017-2022 React 진영에서 표준에 가까웠음.

```tsx
const Button = styled.button`
  background: ${props => props.primary ? "blue" : "gray"};
  padding: 8px 16px;
  &:hover { opacity: 0.8; }
`
```

**매력:** JS 표현식이 스타일에 들어감. props 기반 동적 스타일이 가능. 이전 어떤 CSS 도구로도 못 했음.

**그러나 두 사형선고:**

#### 사형선고 ① — Runtime cost

```
[컴포넌트 렌더]
  ↓ props 변화 감지
  ↓ 템플릿 함수 실행 → CSS 문자열 생성
  ↓ hash 계산 → 클래스명 생성
  ↓ 캐시 조회 → 새 클래스면 <style> 태그 주입
  ↓ DOM 업데이트
```

렌더마다 다 돌아감. 비교 — Tailwind는 className 문자열 그대로 출력, **runtime 0**.

**2022년 Sam Magura 의 글** — Emotion(2위 CSS-in-JS) 메인테이너, Spotify 엔지니어가 *"Why We're Breaking Up with CSS-in-JS"* 발표:
- styled-components가 React 컴포넌트 마운트의 약 50% 오버헤드
- 큰 페이지 TTI 저하 측정 가능
- React 18 concurrent rendering과 충돌 (`useInsertionEffect` 복잡성)
- Streaming SSR 스타일 flush 타이밍 깨짐

이후 신규 프로젝트 채택률 급락.

#### 사형선고 ② — RSC와 근본 충돌

CSS-in-JS는 **runtime에 클라이언트에서** CSS 생성. React Context (테마), DOM API (`<style>` 주입), useState/useEffect 의존. **서버에는 이런 게 없음**.

```tsx
// RSC에서 동작 X
const Button = styled.button`
  background: ${props => props.theme.primary};  // ← Context 필요, 서버 못 씀
`
```

→ Server Component에서 styled-components 사용 불가. `"use client"` 강제 → RSC 핵심 가치(서버 렌더, JS 번들 절감) 사라짐. **CSS-in-JS 쓰는 컴포넌트는 다 클라 컴포넌트가 되는 함정.**

#### 라이브러리들의 운명

| 라이브러리 | 상태 |
|---|---|
| **styled-components** | 2024년 maintenance mode 선언 |
| **Emotion** | 메인테이너 손 뗌 |
| **Stitches** | Modulz 인수 후 discontinued |
| **Next.js 공식 문서** | "App Router에서 CSS-in-JS 비추천" |

#### Zero-Runtime CSS-in-JS — 정신만 살아남음

CSS-in-JS의 DX는 살리되, **빌드 타임에 CSS 추출**:

| 라이브러리 | 핵심 |
|---|---|
| **Vanilla Extract** | TS로 스타일 작성, 빌드 시 정적 CSS 추출. type-safe |
| **Linaria** | styled-components 같은 문법, 빌드 시 추출 |
| **Panda CSS** | styled-system + utility 융합, 빌드 시 추출 |

→ runtime 0 + RSC 호환. 후예들이 살아남았지만 **실행 모델은 정반대**.

---

### CSS Modules의 위치

```tsx
import styles from "./Button.module.css"
<button className={styles.primary}>
```

빌드 시 클래스명 자동 hash (`Button_primary__abc123`) → 글로벌 충돌 X.

**5개 함정 중 해결:**

| 함정 | CSS Modules |
|---|---|
| ① 이름 짓기 | ❌ 여전히 직접 |
| ② 멀어짐 | 🟨 같은 폴더, 다른 파일 |
| ③ Dead CSS | 🟨 컴포넌트 같이 삭제 시 같이 사라짐 |
| ④ Specificity | ✅ 모든 클래스 unique |
| ⑤ 시스템 강제 | ❌ `padding: 17px` 가능 |

**무난한 차선** — 글로벌 네임스페이스 + specificity는 해결. 하지만 시스템 강제 / co-location / 이름 짓기는 Tailwind가 더 강함.

**살아있는 시나리오:**
- 팀이 utility 학습 비용 못 냄
- 매우 커스텀한 keyframe 많음
- 작은 프로젝트 빠른 시작
- 레거시 SCSS 점진 마이그레이션

---

### 탈출구 — `@apply` + Layers

#### `@apply` — utility를 묶어서 클래스로

```css
.btn-primary {
  @apply bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded;
}
```

**언제 쓰나:**
- 마크다운 렌더링 결과물 스타일링
- 외부 라이브러리가 만드는 DOM (Tiptap 에디터 등)
- 컴포넌트 추상화 못 하는 경우

**남용 주의:** utility의 장점(co-location, 시스템 강제, dead CSS)을 다 깎아먹음. **컴포넌트 추상화 가능하면 컴포넌트 우선**, `@apply`는 마지막 수단.

#### Layers — Cascade 순서 제어

```css
@tailwind base;        /* 1. reset, 기본 스타일 */
@tailwind components;  /* 2. 우리 .btn 같은 것 */
@tailwind utilities;   /* 3. bg-blue-500 등 */
```

```css
@layer components {
  .btn-primary { @apply bg-blue-500 px-4 py-2; }
}
```

```html
<!-- bg-red-500이 우선 (utilities가 components보다 뒤 layer) -->
<button class="btn-primary bg-red-500">
```

→ specificity 싸움 없이 **순서로 제어**.

---

### 디자인 토큰 + PostCSS

#### 디자인 토큰 통합

**방향 1.** 토큰을 `tailwind.config.js`에 import:

```ts
// tokens.ts (Figma에서 추출)
export const tokens = {
  colors: { primary: "#3b82f6" },
  spacing: { 1: "4px", 2: "8px" },
}

// tailwind.config.js
import { tokens } from "./tokens"
export default { theme: { colors: tokens.colors, spacing: tokens.spacing } }
```

→ utility(`bg-primary`, `p-2`)가 토큰 참조. JS 객체 하나가 진실의 원천.

**방향 2 (Tailwind v4 권장).** CSS Custom Properties로 양방향:

```css
@theme {
  --color-primary: #3b82f6;
  --spacing-2: 8px;
}
```

런타임 다크모드/테마 전환에 유리 (CSS 변수 값만 바꿈).

#### PostCSS 파이프라인

Tailwind는 PostCSS 플러그인 (v3까지). 빌드 흐름:

```
input.css → Tailwind → autoprefixer → cssnano → output.css
```

**자주 쓰는 플러그인:**

| 플러그인 | 역할 |
|---|---|
| `autoprefixer` | 벤더 prefix 자동 |
| `postcss-nesting` | 중첩 문법 |
| `cssnano` | 프로덕션 압축 |
| `@tailwindcss/typography` | 마크다운/긴 글 자동 (`prose` 클래스) |
| `@tailwindcss/container-queries` | `@container` 지원 |

**Tailwind v4 (2025+):**
- PostCSS 의존성 제거 → 자체 컴파일러 (Rust 기반, Lightning CSS)
- 빌드 속도 10배 ↑
- `tailwind.config.js` 대신 CSS의 `@theme` 디렉티브로 설정

---

### 컴포넌트 Variant 관리 — CVA + tailwind-merge + clsx

#### 문제 — variant가 늘면 className 조립 지옥

```tsx
function Button({ variant, size, disabled }) {
  let className = "rounded font-medium"
  if (variant === "primary") className += " bg-blue-500 text-white hover:bg-blue-600"
  else if (variant === "secondary") className += " bg-gray-200 ..."
  if (size === "sm") className += " px-3 py-1 text-sm"
  // ...if-else 폭발
}
```

#### CVA — Variant 선언적 관리

```tsx
import { cva, type VariantProps } from "class-variance-authority"

const button = cva(
  "rounded font-medium transition-colors",  // 베이스
  {
    variants: {
      variant: {
        primary: "bg-blue-500 text-white hover:bg-blue-600",
        secondary: "bg-gray-200 text-gray-900 hover:bg-gray-300",
        ghost: "bg-transparent text-gray-900 hover:bg-gray-100",
      },
      size: {
        sm: "px-3 py-1 text-sm",
        md: "px-4 py-2 text-base",
        lg: "px-6 py-3 text-lg",
      },
      disabled: { true: "opacity-50 cursor-not-allowed" },
    },
    defaultVariants: { variant: "primary", size: "md" },
    compoundVariants: [
      // primary + lg일 때만 추가
      { variant: "primary", size: "lg", className: "shadow-lg" },
    ],
  }
)

// 타입 자동 추론
type ButtonProps = VariantProps<typeof button> & React.ButtonHTMLAttributes<HTMLButtonElement>

function Button({ variant, size, disabled, className, ...props }: ButtonProps) {
  return <button className={button({ variant, size, disabled, className })} {...props} />
}

<Button variant="primary" size="lg">Click</Button>
```

**가치:**

| | 순진 방식 | CVA |
|---|---|---|
| 코드 양 | if-else 지옥 | 선언적 |
| 타입 | 수동 | `VariantProps` 자동 |
| 조합 (compound) | 중첩 if | `compoundVariants` 선언 |
| 기본값 | 수동 | `defaultVariants` |

#### tailwind-merge — 충돌 자동 해결

Tailwind 함정: 같은 속성 utility가 두 개면 **CSS 순서대로 결정** (의도와 다를 수 있음).

```html
<!-- 의도: px-8로 override. 실제: CSS 순서가 이김 -->
<button class="px-4 py-2 px-8">
```

```ts
import { twMerge } from "tailwind-merge"
twMerge("px-4 py-2 px-8")              // → "py-2 px-8"
twMerge("text-red-500 text-blue-500")   // → "text-blue-500"
```

#### `clsx` — 조건부 조립

```ts
import clsx from "clsx"
clsx("base", isActive && "active", { highlighted: count > 0 })
// → "base active highlighted"
```

#### 표준 조합 — shadcn의 `cn()` 패턴

```ts
// utils.ts (shadcn 표준)
import { clsx, type ClassValue } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

// 사용
<div className={cn("base", isActive && "ring", className)}>
```

**역할 분담:**
- **CVA** — 컴포넌트 variant 시스템 (정의 시점)
- **clsx** — 인스턴스 단위 조건 className (사용 시점)
- **tailwind-merge** — 충돌 해결 (마지막)

#### CVA를 안 쓰는 경우

- variant 1-2개뿐인 단순 컴포넌트 → ternary 충분
- Tailwind 안 쓰는 프로젝트 (CVA는 className 기반)
- 디자인 시스템 자주 바뀌는 초기 (선언 비용 > 즉흥성)

---

### 의사결정 매트릭스

| 도구 | 패러다임 | 2026 추천 |
|---|---|---|
| **Tailwind CSS** | utility-first, 빌드 타임 컴파일 | ✅ 디폴트 |
| **CSS Modules** | scoped CSS, hash | 🟨 Tailwind 못 쓸 때 |
| **CSS-in-JS (runtime)** — styled-components, Emotion | runtime CSS 생성 | ❌ 신규 X |
| **Zero-runtime CSS-in-JS** — Vanilla Extract, Linaria, Panda | 빌드 타임 추출 | 🟨 type-safety 강하게 원할 때 |

**상황별:**

| 상황 | 추천 |
|---|---|
| Next.js App Router (RSC) | **Tailwind** |
| 신규 풀스택 React | **Tailwind + CVA + tailwind-merge + clsx** |
| 팀이 utility 거부 + RSC 안 씀 | **CSS Modules** |
| 강한 TS 안전성 + 디자인 토큰 | **Vanilla Extract** |
| 레거시 styled-components | 점진 마이그레이션 (Tailwind 또는 Vanilla Extract) |

---

## 핵심 질의응답

**Q. 왜 semantic CSS가 무너졌나?**
A. 컴포넌트가 의미 단위가 되면서 CSS class의 "의미" 역할이 사라짐. 그 위에서 5가지 함정(이름 짓기, 멀어짐, dead CSS, specificity, 시스템)이 드러남.

**Q. utility는 더럽지 않나?**
A. 추상화를 컴포넌트로 옮긴 결과. 정의 1번, 사용 N번. co-location + 시스템 강제의 이득이 더 큼.

**Q. utility 라이브러리는 Tachyons 등 선배가 있었는데 왜 Tailwind만 떴나?**
A. 5가지 — 디자인 시스템 커스터마이징 (config), PostCSS/JIT 컴파일러, Variant 시스템 표현력, 컴포넌트 시대 timing, Adam Wathan의 advocacy.

**Q. JIT 이전에는 file size 어떻게?**
A. PurgeCSS 후처리 — 모든 utility 생성(3MB) 후 안 쓰는 거 제거. v3부터 JIT으로 처음부터 쓰는 것만 생성.

**Q. arbitrary values (`w-[762px]`) 의 함정?**
A. JIT가 즉석 생성하지만 디자인 시스템 강제력이 깨짐. 코드 리뷰에서 명시적으로 보이게 표현되어 시스템에서 벗어남이 발견 가능.

**Q. CSS-in-JS는 왜 죽었나?**
A. 두 사형선고 — runtime cost (렌더마다 CSS 생성·hash·DOM 주입), RSC와 근본 충돌 (Context/DOM API 의존, 서버에서 못 씀).

**Q. 그럼 styled-components 매력 (props 기반 동적 스타일) 은 어떻게 대체?**
A. Tailwind: 조건부 className (CVA + clsx). 동적이라기보단 미리 정의된 variant 선택. 진짜 동적 값은 inline style 또는 CSS 변수.

**Q. CSS Modules는 어디 위치?**
A. 5개 함정 중 specificity·충돌만 해결. 시스템 강제 / co-location / 이름 짓기는 못 함. "큰 잘못은 안 하지만 큰 이득도 없는" 차선.

**Q. Container Query는 언제?**
A. 컴포넌트가 여러 위치(사이드바/메인)에 재사용되어 부모 크기에 반응해야 할 때. 페이지 레이아웃은 그대로 미디어쿼리.

**Q. `@apply` 써도 되나?**
A. 외부 마크업 (마크다운, Tiptap 등)이나 컴포넌트 추상화 못 하는 경우만. 컴포넌트로 가능하면 컴포넌트가 우선. 남용하면 utility 장점 다 깎음.

**Q. CVA / tailwind-merge / clsx 역할 차이?**
A. CVA — 정의 시점 variant 시스템. clsx — 사용 시점 조건부 조립. tailwind-merge — 충돌 해결. shadcn의 `cn()`이 이 셋을 묶음.

**Q. Tailwind v4 변화?**
A. PostCSS 의존성 제거, Rust 기반 자체 컴파일러 (Lightning CSS), 빌드 10배 ↑, `@theme` 디렉티브로 설정 (config.js 안 씀).

## 주의사항 / 자주 하는 실수

- **Tailwind 클래스를 동적 문자열로 조립** — `bg-${color}-500` 같이 짜면 JIT가 토큰 추출 못 함 (정규식 스캔 기반). 완전한 클래스명을 명시하거나 CVA의 lookup으로.
- **CSS-in-JS를 RSC에서 사용** — `"use client"` 강제됨. Server Component 가치 사라짐. Tailwind/CSS Modules 또는 zero-runtime CSS-in-JS로.
- **`@apply` 남용** — utility 장점(co-location, 시스템) 다 깎음. 컴포넌트 추상화 가능하면 컴포넌트가 우선.
- **`tailwind-merge` 없이 props로 className override** — 같은 속성 utility 충돌 시 CSS 순서대로 결정. 의도와 다른 결과.
- **arbitrary values 남발** — `w-[17px]` 한두 개는 OK, 많아지면 시스템에서 벗어났다는 신호. 디자인 토큰으로 흡수 검토.
- **mobile-first 반대로 짜기** — `md:hidden lg:block` 같이 desktop 기준으로 깎으려다 모바일 깨짐. baseline은 항상 모바일.
- **Container Query를 페이지 레이아웃에** — viewport 기준이 자연스러운 곳에 컨테이너 쿼리 쓰면 디버깅 어려움. 컴포넌트 내부에만.
- **CVA 없이 큰 컴포넌트의 variant 관리** — if-else 폭발, 타입 안전 X, compound variant 표현 어려움. variant 3개 이상이면 CVA 도입.
- **`tailwind.config.js` 의 `extend` vs 덮어쓰기 혼동** — `theme: { colors: {...} }` 면 기본 색 다 사라짐. `theme: { extend: { colors: {...} } }` 가 보통 의도.

## 참고

- [Tailwind CSS 공식](https://tailwindcss.com/)
- [Tailwind v4 — What's new](https://tailwindcss.com/blog/tailwindcss-v4)
- [class-variance-authority (CVA)](https://cva.style/)
- [tailwind-merge](https://github.com/dcastil/tailwind-merge)
- [clsx](https://github.com/lukeed/clsx)
- [Vanilla Extract](https://vanilla-extract.style/)
- [Linaria](https://linaria.dev/)
- [Panda CSS](https://panda-css.com/)
- [Container Queries (MDN)](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries)
- ["Why We're Breaking Up with CSS-in-JS" — Sam Magura](https://dev.to/srmagura/why-were-breaking-up-wiht-css-in-js-4g9b)
- [Adam Wathan — CSS Utility Classes and Separation of Concerns](https://adamwathan.me/css-utility-classes-and-separation-of-concerns/)
- 관련 학습: [Headless UI 컴포넌트 라이브러리 (Radix/shadcn)](./Headless-UI-라이브러리-비교.md) — 본 문서의 후속, CVA가 shadcn에서 어떻게 쓰이는지
- 관련 학습: [폼 라이브러리 + 검증 비교](../폼-유효성검사/폼-라이브러리-검증-비교.md) — RHF가 styled-components 대신 Tailwind와 결합하는 패턴
