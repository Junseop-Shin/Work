# Headless UI 컴포넌트 라이브러리 비교 (Radix / shadcn vs MUI / Chakra / Mantine)

> 2026-05-08 | RadixUI, shadcn, HeadlessUI, ReactAria, MUI, Chakra, Mantine, a11y, RSC

## 한 줄 요약

UI 컴포넌트 라이브러리는 **"동작 + 스타일을 다 주는 styled 진영"** (MUI/Chakra/Mantine)과 **"동작/a11y만 주고 스타일은 위임하는 headless 진영"** (Radix/Headless UI/React Aria)으로 나뉜다. 2026 표준은 **Radix(동작) + Tailwind(스타일) + shadcn/ui(접착제, 복붙 패러다임) + CVA**의 직교적 조합. styled 진영은 CSS-in-JS 의존 + RSC 충돌로 사내 도구·어드민 같은 디자인 차별화 안 중요한 영역에 후퇴. shadcn의 "라이브러리가 아니라 코드 복사" 발상은 **AI 생태계(v0.dev)와도 자연스럽게 맞물려** 새로운 표준이 됨.

## 핵심 개념

### Headless 패러다임 — 동작과 스타일의 분리

전통적 UI 라이브러리 (MUI, Bootstrap, Ant Design)는 동작 + 스타일 + 마크업을 한 번에 제공:

```tsx
import { Button } from "@mui/material"
<Button variant="contained" color="primary">Click</Button>
```

빠르게 시작하기 좋음. 그러나 **동작과 스타일이 묶여 있음** — 스타일을 바꾸려면 라이브러리의 theme 시스템 안에서만 가능.

Radix는 다름:

```tsx
import * as Dialog from "@radix-ui/react-dialog"

<Dialog.Root>
  <Dialog.Trigger>열기</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay />
    <Dialog.Content>
      <Dialog.Title>제목</Dialog.Title>
      <Dialog.Close>닫기</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

**여기엔 어떤 스타일도 없다** — 글자 크기/색/padding 모두 사용자 책임. 화면에 띄우면 *날 것* 같이 보임.

#### Radix가 제공하는 것

| 영역 | 내용 |
|---|---|
| **키보드 네비** | Esc로 닫기, Tab focus trap, Shift+Tab 역순, 첫 focusable로 자동 포커스, 닫힐 때 트리거로 복귀 |
| **ARIA** | `role="dialog"`, `aria-modal`, `aria-labelledby`, `aria-describedby` 자동 |
| **포커스/스크롤** | body scroll lock, inert 배경, 닫힘 시 lock 해제 |
| **렌더링** | React Portal로 `<body>` 끝에 마운트 (z-index 지옥 방지) |
| **클릭** | 오버레이 클릭 닫기 (옵션), 내부 클릭 무시 |
| **상태 노출** | `data-state="open\|closed"` → CSS 애니메이션 hook |

#### Radix가 제공하지 않는 것

스타일.

이 분리가 headless 패러다임의 본질:
> **동작은 universal + 복잡 → 라이브러리로 통일.**
> **스타일은 service-specific → 사용자에게 위임.**

---

### a11y가 라이브러리화되어야 하는 이유

`Dialog` 한 컴포넌트의 동작 명세를 펼치면 **약 20가지 인터랙션**:

- 키보드 (Esc, Tab, Shift+Tab, 자동 포커스, 포커스 복귀)
- ARIA (role, modal, labelledby, describedby, screen reader announcement)
- 포커스/스크롤 (body lock, inert, 해제)
- 렌더링 (Portal, stacking context 분리)
- 클릭 (외부 클릭, 내부 무시)
- 상태 (data attributes, forceMount)

**직접 짜면 컴포넌트 하나당 며칠.** 게다가 WAI-ARIA 명세가 컴포넌트마다 다름 — Dropdown, Combobox, Tooltip은 모두 명세가 다르고 키보드 동작도 다름.

> Tailwind 진영 격언: *"a11y 라이브러리는 가성비가 끔찍하다 — 만들기 어려운데 안 만들면 시각장애인 사용자가 못 씀."*

도덕적/법적(접근성 법)/제품 품질 비용을 합치면 라이브러리 외 답이 없음.

---

### shadcn/ui — "복붙" 패러다임

Radix는 동작만, 스타일은 매번 직접? shadcn이 그 갭을 메움. **방식이 이전과 완전히 다름**:

```bash
# 일반 라이브러리
npm install @mui/material   # 끝, import해서 쓰면 됨

# shadcn/ui
npx shadcn@latest add button
# → components/ui/button.tsx 가 내 프로젝트에 복사됨
```

**라이브러리를 install하지 않고 코드를 복사**. `package.json`의 dep가 아니라 **내 프로젝트에 들어온 내 파일**.

```tsx
// components/ui/button.tsx — 내 코드, 자유롭게 수정
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center ...",
  {
    variants: {
      variant: { default: "...", destructive: "...", outline: "...", ghost: "..." },
      size: { default: "h-10 px-4 py-2", sm: "h-9 px-3", lg: "h-11 px-8", icon: "h-10 w-10" },
    },
    defaultVariants: { variant: "default", size: "default" },
  }
)

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(...)
```

Radix(dep) 위에 Tailwind + CVA 래퍼만 살짝 입혀놓은 한 파일. **마음대로 수정 가능.**

#### 복붙 패러다임의 이득

**A. 디자인 시스템 즉시 흡수**
```tsx
// shadcn add button → 그 자리에서 회사 토큰으로 수정
const buttonVariants = cva(
  "font-pretendard rounded-brand",  // 회사 폰트/radius
  { variants: { variant: { default: "bg-brand-500 hover:bg-brand-600" } } }
)
```
MUI라면 `theme.palette.primary.main` 우회로만 가능. shadcn은 소스코드 직접 수정이 디폴트.

**B. "라이브러리가 X 기능 제공 안 함" 함정 제거**
MUI에서 "Button에 loading 스피너 위치를 트리거 옆으로 두고 싶다 → MUI는 안에만 둠" → PR 보내거나 wrapping. shadcn은 그냥 그 파일 열어서 수정.

**C. Bundle 사이즈 자유**
```bash
npx shadcn@latest add button   # 만 가져오면 button.tsx + 의존성만
```
MUI는 `@mui/material` 한 덩어리라 tree-shaking 의존. shadcn은 **애초에 import 안 한 컴포넌트는 들어오지도 않음**.

**D. v0 / Registry 생태계**

shadcn 형식이 표준이 되면서:
- **v0.dev** (Vercel) — AI가 shadcn 형식 컴포넌트 생성
- **Origin UI**, **Magic UI**, **aceternity UI** — 호환 컴포넌트 라이브러리
- **Custom Registry** — 회사 내부 컴포넌트도 같은 CLI로 배포

```bash
npx shadcn@latest add https://my-company.com/registry/our-card
```

**CLI 자체가 패키지 매니저화**. 회사 내부 디자인 시스템 배포까지 같은 패턴.

#### 복붙 패러다임의 단점

**A. 업데이트 수동**
```bash
# Radix (dep) — 자동
npm update @radix-ui/react-dialog

# shadcn 컴포넌트 (내 파일) — 수동
npx shadcn@latest diff button       # 변경사항 확인
npx shadcn@latest add button --overwrite  # 덮어쓰기 (커스텀 사라짐)
```
Radix 부분은 자동 업데이트, **shadcn 래퍼 코드의 버그 픽스/개선은 수동 머지**.

**B. "표준"이 아님 — 모든 프로젝트가 다르게 진화**
같은 "shadcn Button"이지만 프로젝트별로 받은 시점·커스텀이 달라 코드가 다름. 회사 차원 통일성을 자동 보장 못 함. 모노레포 + `@company/ui` 내부 패키지로 관리하는 게 답.

**C. 디자인 일관성 자동 강제 X**
MUI는 Material Design 가이드 강제 → 모든 컴포넌트가 같은 spacing/animation. shadcn은 **컴포넌트마다 따로 커스터마이징** → 디자이너가 챙겨야 함.

---

### Styled 진영 — MUI / Chakra / Mantine

| | MUI | Chakra | Mantine |
|---|---|---|---|
| 디자인 철학 | Material Design | 자체 / 중립 | 자체 / 풍부 |
| 스타일링 | Emotion (CSS-in-JS) | Emotion + style props | Emotion + CSS variables |
| 컴포넌트 수 | 가장 많음 | 중간 | 매우 많음 (form, charts, dates 다 내장) |
| Theme 시스템 | createTheme | extendTheme | MantineProvider |
| Headless 옵션 | MUI Base (분리) | Chakra v3 강화 | 내장 X |
| RSC 호환 | 🟨 Pigment CSS (베타) | 🟨 진행 중 | 🟨 v7 부분 지원 |
| 학습 곡선 | 중간 | 낮음 | 중간 |

```tsx
// MUI
<Button variant="contained" sx={{ bgcolor: "primary.main", borderRadius: 2 }}>Click</Button>

// Chakra
<Button colorScheme="blue" size="md" px="4" rounded="md">Click</Button>
```

#### Styled 진영의 RSC 위기

CSS-in-JS의 두 사형선고 (→ [CSS 작성 방식 비교](./CSS-작성방식-비교.md)) 가 그대로 적용:

```tsx
// app/page.tsx (Server Component)
import { Button } from "@mui/material"  // ❌ 못 씀
export default function Page() {
  return <Button>...</Button>  // → "use client" 강제
}
```

→ Server Component 가치(서버 렌더, JS 번들 절감) 사라짐. 대응 시도들:

| 라이브러리 | 시도 |
|---|---|
| MUI v6+ | Pigment CSS (zero-runtime 변환), 베타 |
| Chakra v3 | RSC 호환 + headless 분리 진행 |
| Mantine v7+ | CSS Modules 기반 일부 전환 |

**검증 부족 + 마이그레이션 비용** — 새 프로젝트 안전한 선택은 여전히 Tailwind + Radix.

---

### 비교 매트릭스 — Headless vs Styled

| | Radix + Tailwind + shadcn | MUI / Chakra / Mantine |
|---|---|---|
| 시작 속도 | 중간 (스타일 결정 필요) | 빠름 |
| 디자인 자유도 | 매우 높음 | Theme 시스템 안에서만 |
| RSC 호환 | ✅ | 🟨 어려움 |
| Bundle 사이즈 | 작음 (선택적) | 큼 (트리쉐이킹 의존) |
| 무거운 컴포넌트 (DataGrid 등) | 직접 조립 | 내장 |
| 디자인 일관성 자동 강제 | ❌ (사람이 챙김) | ✅ (시스템이 강제) |
| 디자인 차별화 | ✅ | ❌ |
| 운영/유지보수 | 컴포넌트별 진화 | 라이브러리 업데이트 |

#### 어느 쪽이 나은 시나리오

✅ **Radix + Tailwind + shadcn 추천:**
- Next.js App Router (RSC) + 디자인 차별화
- 강한 브랜드 / 디자인 정체성
- 풀스택 TS, Vercel/Netlify 배포 환경
- 작-중규모 SaaS, B2C 프로덕트
- AI 도구 (v0/Cursor) 친화 환경

✅ **MUI / Chakra / Mantine 추천:**
- B2B 어드민 / 사내 도구 / 대시보드
- DataGrid, DateRangePicker 같은 무거운 컴포넌트가 핵심
- 디자이너 없는 프로젝트, 빠른 MVP
- 대규모 팀 onboarding (개인 디자인 의사결정 ↓)
- Material Design 익숙도 자체가 가치인 도메인

---

### 왜 Radix + Tailwind + shadcn 조합이 표준이 되었나

세 도구가 따로 등장했는데 자연스럽게 **수렴 진화**한 케이스. 각자가 메우는 빈 자리:

```
[Radix Dialog]            ← 동작/a11y
      ↓
[shadcn dialog.tsx]       ← Radix 위에 Tailwind 클래스 입힘
      ↓
[CVA로 variants 정의]      ← size, variant
      ↓
[cn() = clsx + twMerge]   ← 사용 시점 조합
      ↓
[<Dialog>]                ← 사용
```

**5가지 이유:**

**① 직교적 책임 분리**
- Radix는 동작, 절대 스타일 안 줌
- Tailwind는 스타일, 절대 동작 안 줌
- shadcn은 결합, 절대 본인이 라이브러리화 안 함 (복붙)

→ 충돌 없음, 교체 용이.

**② RSC 호환**
- Radix — Server Component 마크업에서 사용 가능 (client 부분만 격리)
- Tailwind — 빌드 타임 정적 CSS, runtime 0
- 자연스럽게 호환

**③ Customization 위치 명확**
| 변경 | 어디서 |
|---|---|
| 동작 | Radix prop |
| 모양 | Tailwind class |
| variant | CVA 객체 |
| 새 컴포넌트 | shadcn registry |

**④ 생태계 합류**
- v0.dev — AI가 shadcn 형식 출력
- Vercel 공식 추천
- Cal.com, Resend, Linear 등 OSS가 shadcn 채택
- Tailwind UI / Origin UI / Magic UI 등 호환 라이브러리

**⑤ "정답" 강제 안 함**
"이걸 다 받아라"가 아니라 **"필요한 것만, 마음대로 수정"** — 실무 현실(회사마다 다른 디자인 시스템)에 맞음.

---

### 다른 Headless 라이브러리

Radix만 있는 건 아님:

| 라이브러리 | 비고 |
|---|---|
| **Radix UI** | 가장 인기, shadcn 표준 |
| **Headless UI** (Tailwind Labs) | Tailwind 자매 프로젝트, 컴포넌트 적음 |
| **React Aria / Aria Components** (Adobe) | 가장 엄격한 a11y, 가파른 학습 곡선 |
| **Ariakit** | Radix와 비슷, 더 가벼움 |
| **Reach UI** | 초기 표준, 현재 deprecated |

**Radix가 이긴 이유:**
- 프리미티브 풍부함 (~30개)
- API 일관성 (Root/Trigger/Content 패턴)
- shadcn의 채택 (생태계 진입)
- WorkOS 인수 후 안정적 지원
- TypeScript 완전 지원

**React Aria** 는 더 정밀하지만 학습 곡선 가파름. 정부 사이트, 의료/금융처럼 a11y 법규가 강한 곳에서 채택. 일반 프로덕트는 Radix.

---

### shadcn의 AI 시대 의미 — 사람이 쓰는 UI에서 AI가 쓰는 UI로

shadcn이 표준이 된 결과로 **AI가 출력해야 하는 컴포넌트 모양이 표준화**됨. 그래서 AI 코드 생성 도구들이 shadcn을 디폴트로 채택:

```
[v0.dev / Cursor / Bolt.new / Lovable]
       ↓ 자연어 입력
       ↓
   shadcn 컴포넌트 조합으로 출력
```

이게 의미하는 바:
- shadcn = 사람이 쓰는 라이브러리 → **AI도 쓰는 라이브러리**로 의미 확장 (2024-2026 트렌드)
- 디자인 시스템이 AI 친화적이 되면 사람-AI 협업 마찰 ↓
- v0가 출력한 코드를 사람이 받아서 수정 → 같은 형식이라 자연스럽게 이어짐

**관련 후속 학습:** JSON-driven UI / AI Generative UI는 별도 학습 슬롯에서 깊이 다룸 (5/31 예정). shadcn의 registry JSON 형식, Vercel AI SDK의 `streamUI` / `streamObject` 패턴, Builder.io / Plasmic 등 server-driven UI 진영 비교.

---

## 핵심 질의응답

**Q. "Headless"가 정확히 무엇을 의미하나?**
A. 동작 + a11y만 제공, 스타일/모양은 제공하지 않음. "머리(스타일)가 없는 컴포넌트"라는 의미.

**Q. 왜 동작과 스타일을 분리하는 게 가치 있나?**
A. 동작은 universal + 복잡 → 라이브러리로 통일하는 게 합리적. 스타일은 service-specific → 사용자 자유로 두는 게 합리적. 한 라이브러리에 묶으면 자유 잃거나 통일성 잃음.

**Q. a11y를 직접 짜면 안 되나?**
A. Dialog 하나만 봐도 키보드 네비, ARIA, 포커스 관리, Portal 등 ~20가지 인터랙션. 컴포넌트마다 며칠 걸림. WAI-ARIA 명세도 컴포넌트마다 달라 도덕적/법적/품질 비용까지 감안하면 라이브러리 외 답 없음.

**Q. shadcn은 "라이브러리"가 아닌가?**
A. 엄밀히 말하면 라이브러리 + CLI 도구. npm 의존성으로 install하는 게 아니라 CLI로 코드를 내 프로젝트에 복사. 결과적으로 "내 코드"가 됨.

**Q. 그럼 shadcn의 단점은?**
A. 업데이트 수동, 통일성 자동 강제 X, 디자인 일관성을 사람이 챙겨야 함. 모노레포 + 내부 패키지로 보완.

**Q. v0.dev가 shadcn에 의존하는 이유?**
A. shadcn이 사실상 표준이라 AI 출력의 "표준 형식"이 됨. 사용자가 받은 코드를 자기 프로젝트에 복사할 때 같은 형식이라 마찰 없음.

**Q. MUI/Chakra/Mantine은 죽었나?**
A. 아니다. 어드민/사내 도구/DataGrid 같은 무거운 컴포넌트 영역에선 여전히 압도적. RSC 안 쓰는 SPA, 디자인 차별화 안 중요한 영역에서 살아남음.

**Q. Radix vs React Aria 차이?**
A. Radix는 일반 프로덕트, React Aria는 더 엄격한 a11y. 정부/의료/금융처럼 법규 강한 곳은 React Aria, 그 외는 Radix가 학습 곡선과 생태계 면에서 우위.

**Q. shadcn 없이 Radix + Tailwind 직접 조합해도 되나?**
A. 가능. shadcn이 하는 일은 "Radix + Tailwind + CVA + cn() 결합 보일러플레이트" 자동화. 직접 짜면 반복 작업 + 디자인 결정을 매번 해야 함.

**Q. 회사 디자인 시스템이 강한 경우 shadcn vs MUI?**
A. 디자인 시스템을 코드로 표현해야 한다면 **shadcn 변형 + 모노레포**. MUI의 theme로는 너무 다른 시스템 표현 어려움. 시스템이 Material에 가까우면 MUI로 바로.

**Q. 생태계가 너무 빨리 변하는데, 안정적 선택은?**
A. 2026 기준 Radix + Tailwind + shadcn은 v0/Cursor/Vercel이 모두 미는 조합이라 향후 3-5년은 안정. MUI는 Pigment CSS 마이그레이션 진행 중이라 단기적 불안정성.

## 추가 (2026-07): Base UI — shadcn의 새 기본값

2026년 7월, **shadcn/ui가 새 프로젝트의 기본 Headless 라이브러리를 Radix에서 [Base UI](https://base-ui.com)로 전환**했다. 위 본문의 "2026 표준 = Radix + Tailwind + shadcn"이라는 전제가 바뀐 지점이다.

Base UI는 **Radix · Floating UI · MUI 팀이 함께 만든 별개 라이브러리**다. Radix의 재배포가 아니다.

### 무엇이 다른가

| | Radix UI | Base UI |
|---|---|---|
| 배포 단위 | 컴포넌트마다 독립 패키지·독립 버전 | 한 패키지(`@base-ui/react`) + subpath export |
| 컴포넌트 수 | ~30 | 40여 개 — Combobox, Autocomplete, NumberField, OTPField, Slider, Progress, Meter, Menubar, NavigationMenu, Drawer, Toolbar 등 포함 |
| 합성 | `asChild` | `render` prop |
| 상태 표현 | `data-state="checked"` (값) | `data-checked` (속성 존재) |
| 팝업 구조 | `Portal > Content` | `Portal > Positioner > Popup` (위치 계산과 표면이 분리) |

shadcn이 든 명분은 성숙도(v1.6.0, 주간 600만 다운로드)와 신규 채택률(Radix 대비 2:1)이지만, **실무에서 체감되는 차이는 컴포넌트 커버리지**다. Radix에 없어서 직접 만들거나 포기하던 것들이 기본 제공된다.

### 마이그레이션에서 실제로 위험한 것

기계적 치환이 불가능하다. shadcn이 codemod가 아니라 **에이전트 skill**로 점진 이전을 제공하는 것이 그 증거다.

진짜 위험은 팝업 구조 변경이 아니라 **스타일 계약이 조용히 깨지는 것**이다. 타입체크로 하나도 안 잡히고 에러도 안 나며 색과 위치만 틀어진다.

- `data-[state=checked]:` → `data-[checked]:` — 값 방식에서 속성 존재 방식으로
- **`disabled:` Tailwind 변형이 죽는다** — Base UI는 Switch/Checkbox 등에서 `<button disabled>` 대신 `<span aria-disabled data-disabled>`를 렌더한다. `:disabled` 의사클래스는 폼 요소에만 걸리므로 영원히 매치되지 않는다. `data-[disabled]:`로 바꿔야 한다
- 같은 이유로 Label의 `peer-disabled:`도 죽는다. 네이티브 폼 요소용과 Base UI용(`peer-data-[disabled]:`)을 **둘 다** 걸어야 한다
- 상태 속성 이름이 컴포넌트마다 다르다 — Tabs.Tab은 `data-active`, Select.Item은 `data-selected`. 한 규칙으로 일반화할 수 없다
- Base UI Separator는 문서 설명과 달리 **role을 붙이지 않는다**. Radix의 `decorative` prop도 없어 접근성 시맨틱을 직접 지정해야 한다

**테스트 환경**: jsdom에 `PointerEvent`가 없어 Base UI 인터랙티브 컴포넌트의 클릭 처리가 크래시한다. setup 파일에 폴리필이 필요하다.

### 판단

- **새 프로젝트** — Base UI. shadcn 기본값이고 커버리지가 넓다.
- **기존 Radix 프로젝트** — 급하지 않다. shadcn도 기존 앱은 마이그레이션 불필요하다고 명시했고 Radix 지원도 계속된다. 다만 Radix에 없는 컴포넌트가 필요해지는 시점이 전환 계기가 된다.
- 전환 시 **유닛 테스트가 유일한 안전망**이 될 수 있다. 위 스타일 계약 파손은 빌드·타입체크·스모크 테스트를 모두 통과한다.

> 실제 적용 사례: `Projects/my-ui-lib` v1.0.0에서 Radix 12개 패키지를 Base UI로 전환하고 신규 컴포넌트 21종을 추가했다.

## 주의사항 / 자주 하는 실수

- **Radix를 "디자인 라이브러리"로 오해** — 스타일 없음. 그대로 쓰면 못생김. 반드시 Tailwind 또는 다른 CSS로 입혀야 함.
- **shadcn 컴포넌트를 라이브러리처럼 취급** — `node_modules`에 있는 게 아니라 `components/ui/`에 있는 내 파일. 자유롭게 수정해도 됨 (그게 디폴트).
- **shadcn 업데이트를 자동으로 기대** — 수동. `npx shadcn@latest diff` 로 확인하고 머지.
- **MUI를 RSC에서 사용** — `"use client"` 강제됨. Server Component 가치 사라짐. 새 프로젝트는 Tailwind+Radix.
- **Radix Portal로 인한 z-index/CSS 문제** — Portal로 body 끝에 마운트되어 부모 stacking context 밖. 부모의 `position: relative` 등이 안 먹음. 의도된 동작.
- **`asChild` prop 무시** — Radix 핵심 패턴. `<Dialog.Trigger asChild><Button>...</Button></Dialog.Trigger>` 처럼 자식에 props 전달. 안 쓰면 중첩 button 등 마크업 오류.
- **a11y 라벨 누락** — `<Dialog.Title>`, `aria-label` 등 빠뜨리면 screen reader 침묵. Radix는 dev 모드에서 경고하지만 기억해야 함.
- **shadcn을 모놀리식으로 다 받기** — `npx shadcn@latest add button` 처럼 필요한 것만. 다 받으면 코드 비대화.
- **Headless UI(Tailwind Labs)와 React Aria(Adobe) 혼동** — 이름 비슷하지만 다른 라이브러리. Radix 진영도 "headless"로 부름.

## 참고

- [Radix UI 공식](https://www.radix-ui.com/)
- [Base UI 공식](https://base-ui.com/)
- [shadcn/ui 공식](https://ui.shadcn.com/)
- [v0.dev](https://v0.dev/)
- [Headless UI (Tailwind Labs)](https://headlessui.com/)
- [React Aria / Aria Components](https://react-spectrum.adobe.com/react-aria/)
- [Ariakit](https://ariakit.org/)
- [MUI](https://mui.com/)
- [Chakra UI](https://chakra-ui.com/)
- [Mantine](https://mantine.dev/)
- [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)
- 관련 학습: [CSS 작성 방식 비교](./CSS-작성방식-비교.md) — Tailwind / CVA / tailwind-merge / clsx — 본 문서의 전제
- 관련 학습 (예정): JSON-driven UI / AI Generative UI — shadcn registry 형식, Vercel AI SDK, server-driven UI 진영 (5/31 슬롯)
- 관련 학습: [폼 라이브러리 + 검증 비교](../폼-유효성검사/폼-라이브러리-검증-비교.md) — RHF + zodResolver + shadcn Form 결합 패턴
