# 트랙 C — my-ui-lib Base UI 전환 + 툴체인 최신화

> **완료 (2026-07-29).** [PR #23](https://github.com/Junseop-Shin/my-ui-lib/pull/23) 머지, `1.0.0` 배포됨.
> 커밋 26개. 선행 작업으로 [PR #21](https://github.com/Junseop-Shin/my-ui-lib/pull/21)(analytics 정리),
> [PR #22](https://github.com/Junseop-Shin/my-ui-lib/pull/22)(CI 게이트 + minor).
>
> | | 착수 전 | 완료 |
> |---|---|---|
> | UI 의존성 | Radix 12개 + sonner | `@base-ui/react` 1개 |
> | 컴포넌트 | 33 | 57 (신규 21 · 대체 3) |
> | vitest | 74 | 80 |
> | Storybook 스위트 / 테스트 | 30 / 76 | 41 / 109 |
> | `dist/*.d.ts` | 40 | 61 |
> | 툴체인 | TS 5.7 · Vite 6 · Storybook 8 | TS 6 · Vite 8 · Storybook 10 · Vitest 4 · ESLint 10 |
>
> **육안 확인에서 잡힌 버그 2건** (스모크 테스트를 전부 통과하던 것들):
> 1. 스토리에 남아 있던 Radix `asChild` → `<button>` 중첩으로 드롭다운이 열리지 않음. `render` prop으로 교체(8곳). 추적 중 `GroupLabel` 컨텍스트 에러, `side`/`align` 전달 누락, Tooltip `role` 누락도 함께 발견
> 2. Storybook docs iframe의 `<base target="_parent">` 때문에 스토리 내 `href="#"` 클릭 시 Storybook UI가 iframe URL로 탈출. `preview-head.html`에서 스토리 범위만 차단

## Context

[shadcn/ui가 2026년 7월부터 기본 컴포넌트 라이브러리를 Radix에서 Base UI로 전환](https://news.hada.io/topic?id=31163)했다. Base UI는 v1.6.0, 주간 다운로드 600만 이상이고 신규 선택이 Radix 대비 2:1이다. 기존 앱은 마이그레이션이 강제되지 않는다.

`my-ui-lib`은 학습·포트폴리오 목적의 디자인 시스템이므로 **생태계 기본값을 따라가는 것 자체가 목적**이다. 추가로 [트랙 D](./트랙D-html-편집-익스텐션.md)가 my-ui-lib에서 정적 스니펫을 뽑아 소비할 예정이라, 그 전에 컴포넌트 내부 구현이 안정화돼 있어야 한다. 전환 후에 스니펫을 뽑아야 두 번 뽑지 않는다.

### 무엇이 실제로 다른가

"기능 뼈대만 제공한다(headless·unstyled)"는 개념은 둘이 같다. 갈리는 지점은 셋이다.

**1. 배포 단위** — Radix는 컴포넌트마다 독립 패키지·독립 버전이고, Base UI는 한 패키지에 subpath export다.
```
Radix     @radix-ui/react-avatar ^1.1.11 · react-checkbox ^1.3.3 · react-dialog ^1.1.15 …
          → 12개 패키지, 12개 버전 라인
Base UI   @base-ui/react 1.6.0
          → 1개 패키지, subpath export 40여 개
```
업그레이드가 12줄에서 1줄이 되고, 내부 공통 의존성(`react-primitive`, `react-context` 등)의 버전이 갈려 번들에 중복 적재되는 문제가 구조적으로 사라진다.

**2. 컴포넌트 커버리지** — Base UI는 40여 개로 Combobox, Autocomplete, NumberField, OTPField, Menubar, NavigationMenu, Drawer, Toolbar 등 Radix에 없는 것들을 포함한다. 라이브러리를 확장할 때 1번보다 크게 체감될 차이다.

**3. API 표면** — 같지 않다. 하위 파트 구성과 합성 방식이 다르다 (아래 "왜 2벌로 가지 않는가" 참고).

### 전환이 급한 일은 아니다

shadcn이 든 근거는 Base UI의 성숙도(v1.6.0, 주간 600만 다운로드)와 신규 채택이 2:1로 앞선다는 점이지 위 1·2번이 아니다. 기존 앱은 마이그레이션이 불필요하고 Radix 지원도 계속된다.

즉 이건 **고장나서 고치는 전환이 아니라 기본값을 따라가는 전환**이다. 학습·포트폴리오 레포에서는 그 자체로 타당한 이유지만, 급하지 않다는 사실은 기록해 둔다 — 일정에 밀리면 미뤄도 되는 작업이다.

**목표**: `@radix-ui/*` 의존성 12개를 `@base-ui/react`로 교체하고, 이어서 툴체인을 최신화한다. 두 작업 모두 **시각적·동작적 회귀 없이** 끝낸다.

## 현황

`Projects/my-ui-lib` — `@junseop-shin/my-ui-lib` v0.1.0

- Atomic design: `src/components/atoms` (17) · `molecules` (8) · `organisms` (7)
- Radix 사용 파일 12개

### 검증 수단 실태 (Phase 0에서 확인·수정)

착수 시점에 안전망이 연결돼 있지 않았다. 통과하는 테스트 150개가 아무 데서도 실행되지 않고 있었다.

| 수단 | 착수 전 | Phase 0 이후 |
|---|---|---|
| `npm run build` (tsc + vite) | ✅ CI 연결됨 | ✅ |
| vitest — 74 tests | 동작하나 **스크립트조차 없음** | ✅ `test:unit`, CI 연결 |
| test-storybook — 76 tests | 동작하나 CI 없음 | ✅ `test:storybook:ci`, CI 연결 |
| Chromatic | 스크립트 플래그 오류, CI 없음, 토큰 없음 | ❌ **폐기** |

**Chromatic은 쓰지 않는다.** 프로젝트에 컴포넌트 7개만 등록돼 있어 베이스라인으로 기능하지 못하는 상태였다. 토큰을 등록해도 의미 있는 비교가 되지 않는다.

**그 결과 픽셀 단위 시각 회귀를 잡는 자동 수단이 없다.** Phase 3에서 `Portal > Positioner > Popup` 구조로 바뀌면 클래스가 붙던 노드가 이동하는데, 타입체크도 vitest도 이를 검출하지 못한다. 해당 단계는 **Storybook을 직접 띄워 사전/사후를 눈으로 비교하고 스크린샷으로 근거를 남긴다.** 이것이 이 계획에서 가장 약한 고리이므로 기록해 둔다.

### 툴체인 격차 (2026-07-29 확인)

| 패키지 | 현재 | 최신 | 격차 |
|---|---|---|---|
| react / react-dom | 19.0.0 | 19.2.8 | minor |
| tailwindcss | 4.1.3 | 4.3.3 | minor |
| @vitejs/plugin-react-swc | 3.8.0 | 4.3.2 | **major 1** |
| vitest | 3.2.4 | 4.1.10 | **major 1** |
| eslint | 9.21.0 | 10.8.0 | **major 1** |
| vite-plugin-dts | 4.5.4 | 5.0.3 | **major 1** |
| vite | 6.2.0 | 8.1.5 | **major 2** |
| typescript | 5.7.2 | 7.0.2 | **major 2** |
| storybook | 8.6.12 | 10.5.5 | **major 2** |

TypeScript 7과 Storybook 10은 각각 그 자체로 별도 프로젝트 규모다. minor와 major를 한 묶음으로 다루지 않는다.

### 의존성 정리 대상 (별도 발견)

- **`@faker-js/faker`가 `dependencies`에 있다.** 배포되는 라이브러리이므로 소비처가 faker를 함께 받는다. `devDependencies`로 옮긴다. 스토리에서만 쓰이는지 먼저 확인할 것 — 런타임 코드가 참조하고 있다면 그 참조부터 걷어내야 한다.
- **`next ^16.1.0`이 `devDependencies`에 있다.** React 컴포넌트 라이브러리에 Next가 필요한 이유가 보이지 않는다. 실제 참조를 확인하고 없으면 제거.

## 대응표 (Base UI v1.6 기준, 확인 완료)

> **초판의 이 표는 틀렸다.** 패키지 이름만 대응시켰을 뿐 API 형태를 확인하지 않아 7개를 "1:1"로 잘못 분류했다. 실제로 구현하며 확인한 결과로 아래를 대체한다.

**A군 — 실제로 가까운 것 (4개, 완료)**

| Radix | Base UI | 파일 | 실제로 바뀐 것 |
|---|---|---|---|
| `react-label` | 없음 → 네이티브 `<label>` | `atoms/Label.tsx` | Form이 자체 필드 컨텍스트를 가져 `Field` 불필요 |
| `react-separator` | `separator` | `atoms/Separator.tsx` | `decorative` 없음, **role을 아예 안 붙임** → 직접 지정 |
| `react-avatar` | `avatar` | `atoms/Avatar.tsx` | import·displayName만 |
| `react-switch` | `switch` | `atoms/Switch.tsx` | `<span>` 렌더 → 스타일 계약 3곳 파손 |
| `react-checkbox` | `checkbox` | `atoms/Checkbox.tsx` | Switch와 동일 |

**B군 — 하위 파트 구성이 다름 (7개)**

| Radix | Base UI | 파일 | 구조 차이 |
|---|---|---|---|
| `react-tooltip` | `tooltip` | `atoms/Tooltip.tsx` | `Content` → `Positioner` + `Popup`. `sideOffset`이 Positioner로 이동 |
| `react-tabs` | `tabs` | `molecules/Tabs.tsx` | `Trigger` → `Tab`, `Content` → `Panel`, `Indicator` 신설 |
| `react-scroll-area` | `scroll-area` | `atoms/ScrollArea.tsx` | Viewport 안에 `Content` 계층 추가 |
| `react-radio-group` | `radio-group` + `radio` | `atoms/RadioGroup.tsx` | 네임스페이스 2개로 분리 |
| `react-select` | `select` | `atoms/Select.tsx` | `Portal > Positioner > Popup > List > Item` |
| `react-dialog` | `dialog` | `atoms/Dialog.tsx` | `Portal > Backdrop > Popup` |
| `react-dropdown-menu` | `menu` | `molecules/DropdownMenu.tsx` | 이름 변경 + Positioner 계층 |

## 스타일 계약 파손 — 이 전환의 진짜 위험

Base UI는 Radix와 **상태 표현 방식이 다르다.** 타입체크로는 하나도 잡히지 않고, 에러 없이 색·위치만 틀어진다.

| Radix | Base UI | 영향 |
|---|---|---|
| `data-state="checked"` (값) | `data-checked` (속성 존재) | `data-[state=checked]:` 셀렉터 전멸 |
| `<button disabled>` | `<span aria-disabled data-disabled>` | **`disabled:` Tailwind 변형이 영원히 매치 안 됨** |
| 위와 같음 | 위와 같음 | Label의 `peer-disabled:` 도 함께 사망 |
| `role="separator"` 자동 | role 없음 | 접근성 시맨틱 소실 |

`disabled:opacity-50`이 조용히 죽는 게 가장 위험하다. 컴포넌트마다 `data-[disabled]:`로 바꾸고, Label에는 네이티브 폼 요소용 `peer-disabled:`와 Base UI용 `peer-data-[disabled]:`를 **둘 다** 걸어야 한다.

## jsdom에 PointerEvent가 없다

Base UI 인터랙티브 컴포넌트는 클릭 처리에 `PointerEvent`를 쓰는데 jsdom이 이를 구현하지 않아 테스트가 크래시한다. `vitest.setup.ts`에 최소 폴리필을 넣어 해결했다.

## Label — Field 없이 네이티브로 (완료)

당초 "아톰은 네이티브, 필드 라벨은 `Field.Label`" 두 갈래로 계획했으나, **실제 코드를 보니 갈래가 하나로 줄었다.**

`molecules/Form.tsx`는 이미 `FormItemContext` + `useFormField`로 자체 필드 컨텍스트를 갖고 있고, `Form.Label`이 `htmlFor={formItemId}`로 연결을 직접 처리한다. Base UI `Field`를 도입하려면 이 손수 만든 컨텍스트를 통째로 걷어내야 하는데, 그건 명시적으로 제외한 "폼 아키텍처 재설계"다.

**결론: `Label`만 네이티브 `<label>`로 내리고 `Form.tsx`는 손대지 않는다.** Radix Label이 `<label>`에 더하던 동작은 더블클릭 텍스트 선택 방지 하나뿐이라 그대로 보존했다. 공개 props 변경 없음.

`react-hook-form`도 그대로 둔다.

## 왜 2벌(Radix판 + Base UI판)로 가지 않는가

두 구현을 나란히 유지하는 안을 검토했고, 하지 않기로 한다.

**전제부터 틀렸다.** "Base UI는 Radix 그대로인데 배포처만 다르다"가 아니다. Select 구조를 비교하면:

```
Radix     Root > Trigger > Portal > Content > Viewport > Item
Base UI   Root > Trigger > Portal > Positioner > Popup > List > Item
```

`Positioner`가 한 겹 더 있고 `Content`가 `Popup`으로 갈렸다. 합성 방식도 `asChild`(Radix)가 아니라 `render` prop이다. Radix·Floating UI·MUI 팀이 *함께 만든* 별개 라이브러리이지 Radix의 재배포가 아니다. shadcn이 codemod가 아니라 에이전트 skill로 점진 이전을 제공하는 것도 기계적 치환이 불가능하다는 증거다.

즉 2벌은 "같은 코드 두 벌"이 아니라 **다른 구현 두 벌**이고, 유지비가 실제로 발생한다 — Radix 사용 파일 12개 이중 관리, 스토리·테스트·Chromatic 스냅샷 2배(스냅샷은 과금 단위), 새 컴포넌트마다 두 번 작성.

**그리고 2벌이 사려는 것은 패키지 버전이 이미 해준다.** 이 라이브러리는 GitHub Packages로 배포되므로 Radix판은 `0.1.x`로 레지스트리에 남고, Base UI판을 `1.0.0`으로 올리면 소비처가 원하는 쪽을 고르면 된다. 소스를 두 벌 유지할 필요가 없다.

`.github/workflows/publish.yml`이 main 푸시 시 자동 배포하되 **버전이 이미 배포된 것과 같으면 건너뛴다.** 즉 버전 범프 시점이 곧 배포 시점이다.

**대신:**
- 전환 완료 시 **`1.0.0`으로 메이저 범프**. 공개 props를 유지하더라도 내부 DOM 구조와 클래스 적용 지점이 바뀌므로, 소비처가 클래스로 오버라이드하고 있으면 깨진다 — minor로 넘길 변경이 아니다. `0.1.x`(Radix판)는 레지스트리에 그대로 남아 소비 가능
- 전환 직전 커밋에 `v0.1.0-radix` 태그를 박아 소스 기준점도 남긴다
- 비교 학습이 목적이라면 `Study/기타/UI-스타일링/Headless-UI-라이브러리-비교.md`에 Base UI 항목을 추가하고 전환 경험을 쓴다 — 2벌 유지보다 싸고 포트폴리오 가치도 높다

## 진행 순서

**Base UI 전환이 먼저, 메이저 툴체인이 나중이다.** 지금 Chromatic 베이스라인이 살아 있고 검증된 상태이므로, 안전망이 가장 튼튼할 때 시각적으로 위험한 작업(Base UI 전환)을 끝낸다. Storybook 10 업그레이드는 베이스라인을 리셋시킬 가능성이 높아서, 순서를 뒤집으면 안전망이 약해진 상태로 전환하게 된다.

**한 컴포넌트 = 한 커밋**, 각 커밋마다 Chromatic diff를 통과시킨다.

### Phase 0 — 기준선 고정 + 안전한 minor
브랜치 `chore/deps-minor`

- `npm run build` · `npm run test`(test-storybook) 통과 확인
- Chromatic 베이스라인 스냅샷 확정 — 이 시점 스냅샷이 회귀 판정 기준이다
- `v0.1.0-radix` 태그 생성
- minor·patch만 올린다: `react`/`react-dom` 19.2.8, `@types/react*`, `tailwindcss` 4.3.3
- Chromatic diff 확인 후 베이스라인 재확정

메이저는 이 단계에 넣지 않는다.

### Phase 1 — Label 정리
아톰은 네이티브 `<label>` 래퍼로, `Form.tsx`의 필드 라벨은 `Field.Label`로. 공개 props 유지.

### Phase 2 — 1:1 대응 컴포넌트 교체
브랜치 `chore/base-ui-migration`. 의존성 낮은 순서로:
```
Separator → Avatar → Switch → Checkbox → Tooltip → ScrollArea → Tabs
```
각 단계: 교체 → 스토리 확인 → 테스트 → Chromatic diff 검토 → 커밋

### Phase 3 — 구조가 다른 컴포넌트
하위 파트 구성이 달라 Tailwind 클래스 적용 지점이 이동한다:
```
RadioGroup → Select → Dialog → DropdownMenu
```

### Phase 4 — 없음

당초 "전환 마무리 + 1.0.0 배포" 단계를 두었으나 해체했다.

- **의존성 제거**는 별도 단계가 아니라 Phase 1~3에서 컴포넌트를 옮길 때마다 해당 `@radix-ui/*`를 함께 지우는 것이 맞다. 그렇게 진행했고 12개 전부 제거됐다.
- **문서 갱신**은 Phase 5(툴체인)와 6(신규 컴포넌트)이 README·INTERVIEW의 서술 대상을 또 바꾸므로, 지금 쓰면 두 번 쓴다. Phase 7로 옮긴다.
- **버전 범프·배포**도 같은 이유로 Phase 7. 여기서 1.0.0을 내면 곧바로 1.1.0·1.2.0이 따라붙는다.

### Phase 5 — 메이저 툴체인 + 의존성 정리
메이저마다 커밋을 나눈다. 한 커밋에 여러 메이저를 섞으면 무엇이 깨뜨렸는지 알 수 없다. (브랜치 분리는 하지 않기로 함 — 트랙 C 전체를 한 PR로 낸다.)

> **계획한 순서와 목표 버전이 둘 다 틀렸다.** peer 의존성 그래프가 순서를 강제했고, 최신 버전에 도달할 수 없는 항목이 있었다. 실제 진행 결과로 대체한다.

```
5-1  의존성 정리     faker · next 제거 (둘 다 참조 0건 — 이동이 아니라 완전 제거)
5-2  typescript      5.7 → 6.0.3      ⚠ 7.0.2 불가
5-3  eslint          9 → 10           + typescript-eslint, react-hooks(5→7), react-refresh
5-4  storybook       8 → 10           ⚠ 마지막이 아니라 여기로 앞당김
5-5  vite            6 → 8            + plugin-react-swc 4, vite-plugin-dts 5
5-6  vitest          3 → 4
```

**TypeScript 7에 갈 수 없다.** `typescript-eslint` 최신(8.65.0)의 peer가 `typescript >=4.8.4 <6.1.0`이다. 강행하면 타입 인식 린팅을 통째로 잃는다. **최신화의 목표는 "최신 버전"이 아니라 "생태계가 함께 설 수 있는 최고 버전"** — 6.0.3이다.

**Storybook을 마지막에 둘 수 없다.** `@storybook/react-vite@8`의 peer가 `vite ≤6`이라, Storybook을 먼저 올리지 않으면 Vite 8이 설치되지 않는다. "Storybook이 깨지면 검증 수단이 전부 사라진다"는 우려로 뒤에 뒀었지만 peer 그래프가 반대를 요구했고, 실제로는 문제 없이 올라갔다.

### 눈에 안 보이는 파손 하나

`vite-plugin-dts` 5가 **`dist/index.d.ts`를 `export {}` 한 줄만 내보냈다.** `package.json`의 `types`가 이 파일을 가리키므로 소비처가 타입을 하나도 받지 못하는 상태였다.

원인은 tsconfig 해석 변경이다. 루트 `tsconfig.json`은 `files: []`에 references만 있어 대상 파일이 0개인데 v5는 이를 그대로 따른다. `tsconfigPath: './tsconfig.app.json'` 지정으로 복구(`.d.ts` 40개).

**빌드도 테스트도 Storybook도 이걸 잡지 못한다.** 계획에 넣어둔 "`dist/` 산출물 직접 확인"이 유일한 검출 수단이었다.

### 그 외 발견

- `@testing-library/user-event`가 테스트 18개 파일에서 직접 import되는데 `package.json`에 없었다. 제거한 `@storybook/test`의 전이 의존성으로 딸려오던 것이 Vitest 4 설치 때 드러났다. 명시적 devDependency로 추가.
- UMD 전역 이름 미지정으로 Rollup이 추측하고 있었고 버전이 바뀌며 결과가 달라졌다(`lucideReact` → `lucide_react`). `output.globals`에 명시.
- 기존 린트 오류(`any`, 미사용 import, 빈 인터페이스)는 업그레이드 전부터 있던 것이라 그대로 뒀다. `npm run lint`는 CI에 없다.
- Storybook 업그레이드 CLI가 레거시 패키지(`@storybook/blocks`·`test`·`jest`·`testing-library`, 참조 0건)를 남기고 `@storybook/addon-mcp`를 임의로 추가했다. 모두 정리. Chromatic 패키지·설정도 함께 제거.

### Phase 6 — Base UI 신규 컴포넌트 추가 (`1.1.0`)

Base UI는 40여 개로 Radix보다 커버리지가 넓다. 전환의 부수 효과가 아니라 **전환을 하는 이유 중 하나**이므로 계획에 포함한다.

my-ui-lib에 없고 Base UI에 있는 것 — 금융권 디자인 시스템 관점에서 값이 큰 순:

| 컴포넌트 | 쓰임 |
|---|---|
| `number-field` | 금액·수량 입력. 증감 버튼·포맷·범위 제약 |
| `combobox` / `autocomplete` | 종목·계좌 검색 |
| `slider` | 비중·배분 조절 |
| `progress` / `meter` | 진행률·게이지 |
| `popover` / `preview-card` | 지표 설명 툴팁보다 무거운 보조 정보 |
| `accordion` / `collapsible` | 상세 내역 접기 |
| `drawer` | 모바일 시트 |
| `toolbar` / `menubar` / `navigation-menu` | 차트·대시보드 상단 조작 |
| `otp-field` | 2FA 입력 |
| `toggle` / `toggle-group` | 기간·차트 유형 전환 |
| `alert-dialog` | 주문 확인처럼 되돌릴 수 없는 동작 |
| `context-menu` | 차트 우클릭 |

**Phase 5보다 뒤에 둔다.** Storybook 10 마이그레이션 비용은 스토리 개수에 비례하므로, 컴포넌트를 늘리기 전에 툴체인을 올리는 쪽이 싸다.

무엇을 몇 개 만들지는 착수 시점에 정한다. 전부 만들 이유는 없다 — 실제로 쓸 자리가 있는 것부터.

### Phase 7 — 문서 갱신 + `1.0.0` 배포

**문서 갱신** — 이 시점에 라이브러리의 최종 모습이 확정되므로 한 번만 쓴다.
- `README.md` — Radix 서술 → Base UI, 컴포넌트 목록에 Phase 6 신규분 반영
- `INTERVIEW.md` — "Radix UI를 왜 썼나요?" 항목을 Base UI 기준으로 다시 쓰고, 전환 과정에서 겪은 스타일 계약 파손을 사례로 넣으면 내용이 강해진다
- `Study/기타/UI-스타일링/Headless-UI-라이브러리-비교.md`에 Base UI 항목 추가

**배포** — Base UI 전환(1~3) + 최신 툴체인(5) + 확장된 컴포넌트 세트(6)를 한 번에 담아 메이저 범프.

`0.1.x`(Radix판)는 레지스트리에 그대로 남아 계속 소비 가능하다. main 머지 시 `publish.yml`이 자동 배포하며, 버전이 이미 배포된 것과 같으면 건너뛴다.

## 검증

### Phase 0~4 (Base UI 전환)

| 항목 | 방법 | 통과 기준 |
|---|---|---|
| 빌드 | `npm run build` | 성공, `dist/` 산출물 정상 |
| 동작 | `npm run test` (test-storybook) | 기존 테스트 전량 통과 |
| 시각 | Chromatic diff | 의도한 변경만. 예상 못 한 diff는 전부 조사 |
| 접근성 | `@storybook/addon-a11y` | 경고 수가 베이스라인 대비 증가하지 않음 |

Chromatic이 이미 붙어 있는 게 이 전환의 핵심 자산이다. 시각 회귀는 코드 리뷰로 못 잡는데, 픽셀 diff는 잡는다.

### Phase 5 (툴체인)

각 메이저마다 위 4개 항목을 그대로 돌린다. 추가로:

| 항목 | 방법 | 통과 기준 |
|---|---|---|
| 타입 | `tsc -b` | 새 에러 0. TS 7에서 늘어난 검사로 에러가 뜨면 **끄지 말고 고친다** |
| 배포 산출물 | `dist/` diff | ESM·UMD·`.d.ts` 3종이 이전과 동등하게 나오는가 |
| 소비 | 실제 소비처에서 `1.0.x` 설치 후 빌드 | 성공 |

`dist/` 확인이 특히 중요하다. Vite 8과 vite-plugin-dts 5는 번들 출력과 타입 선언 생성 방식을 바꿀 수 있는데, 이건 Chromatic도 테스트도 잡지 못한다.

## 리스크

1. **Portal/Positioner 구조 차이 (확인됨).** Base UI Select는 `Portal > Positioner > Popup > List > Item`으로, Radix의 `Portal > Content > Viewport > Item`보다 한 겹 깊다. 클래스를 붙이던 노드가 사라지거나 이동한다. Menu·Tooltip도 같은 패턴. → Chromatic diff로 잡는다. 3단계를 뒤로 뺀 이유다.
   - 합성도 `asChild` → `render` prop으로 바뀐다. `asChild`를 쓰는 자리를 미리 전수 조사해 둘 것.
2. **data-attribute 이름 차이.** Tailwind v4에서 `data-[state=open]` 같은 셀렉터로 상태 스타일을 걸었다면, Base UI의 속성명이 다를 수 있다. 3단계 컴포넌트에서 특히 확인.
3. **shadcn 마이그레이션 skill.** shadcn이 codemod 대신 에이전트 skill로 점진 이전 경로를 제공한다. 쓸 수 있으면 쓰되, 이 라이브러리는 shadcn CLI로 생성한 게 아니라 직접 작성한 컴포넌트이므로 **그대로 적용되지 않을 수 있다.** 참고 자료로 두고 수동 전환을 기본으로 잡는다.
4. ~~Storybook 10이 Chromatic 베이스라인을 리셋할 수 있다~~ — Chromatic을 쓰지 않기로 해 무효.
5. ~~TypeScript 7은 Go 기반 재구현이다~~ — TS 7에 가지 못했으므로 무효. 대신 TS 6에서는 `baseUrl` 옵션이 제거돼 `tsconfig.app.json`에서 삭제해야 했다(`paths`는 tsconfig 위치 기준으로 해석되므로 동작 동일).

## 범위 밖

- 폼 아키텍처 변경 (react-hook-form 유지)
- Base UI 신규 컴포넌트 도입 (Combobox, Toast 등) — 전환이 끝난 뒤 별건
- `sonner` → Base UI `toast` 교체 — 현재 잘 동작하므로 건드리지 않음
- 그 외 의존성(`motion`, `recharts`, `@tanstack/react-table`, `react-hook-form`, `date-fns`)의 메이저 업그레이드 — Phase 5는 툴체인만 다룬다. 런타임 라이브러리는 별건
