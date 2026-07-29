# 후속 — my-ui-lib v1.0.0을 profile에 반영

## Context

[트랙 C](./트랙C-base-ui-전환.md)로 my-ui-lib이 v1.0.0이 되면서 Radix → Base UI 전환, 컴포넌트 33 → 57, 툴체인 5종 메이저 업그레이드가 이뤄졌다. 이를 `Projects/profile`에 반영한다.

## 먼저 — 코드 마이그레이션은 할 일이 없다

착수 전 조사에서 **profile이 my-ui-lib을 실제로 소비하지 않는다**는 것이 확인됐다.

| 대상 | 상태 |
|---|---|
| 루트 Vite 앱 (`src/`, ts·tsx 36개) | `package.json`에 `@junseop-shin/my-ui-lib: ^0.1.0` 선언, **import 0건** |
| `next/` 앱 (Next 16, 별도 package.json) | my-ui-lib 의존성 자체가 없음 |
| `next/data/projects.ts` | 포트폴리오 **소개 항목**으로만 등장 |

`^0.1.0`은 `<1.0.0`이라 1.0.0을 자동으로 받지도 않는다. **버전을 올려도 바뀌는 코드가 없으므로 올리지 않는다.** 실제로 컴포넌트를 쓸 자리가 생기면 그때 `^1.0.0`으로 넣는다.

---

## 할 일

### 1. 포트폴리오 항목 갱신 — 가장 실질적

`next/data/projects.ts`의 `slug: "my-ui-lib"` 항목이 v0.1.0 시절 내용이다.

**현재**
```ts
stack: ["React 19", "TypeScript", "TailwindCSS", "Radix UI", "Rollup"],
sections: [{ heading: "주요 특징", items: [
  "Light / Dark / Finance 3가지 테마 (CSS 변수 커스터마이징)",
  "ThemeProvider 기반 전역 테마 관리",
  "Radix UI 기반 접근성 준수 컴포넌트",     // ← 사실과 다름
  "npm 패키지 배포 (@junseop-shin/my-ui-lib)",
]}]
```

**고칠 것**
- `stack`의 `"Radix UI"` → `"Base UI"`
- `"Radix UI 기반 접근성 준수 컴포넌트"` → Base UI 기준으로 다시 쓰기
- 없는 내용 추가 — 컴포넌트 57종, Storybook 41스위트/109테스트, v1.0.0

**여기서 전환 경험 자체가 포트폴리오 재료다.** 단순히 "Base UI 씀"보다 아래가 훨씬 강하다.

- Radix 12개 패키지 + sonner → `@base-ui/react` 1개로 정리
- 마이그레이션에서 **타입체크로 안 잡히는 스타일 계약 파손**을 기존 테스트로 검출한 경험 (`disabled:`가 조용히 죽는 문제)
- 통과하는 테스트 150개가 CI에서 실행되지 않고 있던 것을 발견해 게이트를 먼저 연결한 것
- `vite-plugin-dts` 업그레이드로 타입 선언이 통째로 사라진 것을 `dist/` 직접 확인으로 잡은 것

상세 근거는 my-ui-lib의 `INTERVIEW.md`에 이미 정리돼 있으므로 거기서 가져온다.

**검증**: `next/` 앱 빌드 통과 + 로컬에서 `/projects/my-ui-lib` 페이지 확인.

### 2. 미사용 의존성 제거

루트 `package.json`의 `@junseop-shin/my-ui-lib: ^0.1.0`은 import 0건이다. my-ui-lib에서 `@faker-js/faker`·`next`를 지운 것과 같은 성격.

**단, 루트 Vite 앱이 아직 살아 있는지 먼저 확인한다.** `firebase.json`이 `dist`를 public으로 가리키고 있어 배포 대상일 가능성이 있다. 죽은 앱이면 3번으로 넘어간다.

**검증**: 제거 후 `npm run build` 통과.

### 3. 루트 Vite 앱과 `next/` 앱의 관계 정리 — 판단 필요

한 레포에 두 앱이 있다.

```
profile/
├── src/          Vite + React Router 앱 (TS 5.7 · Vite 6 · ESLint 9)
├── next/         Next 16 앱 — handoff.md 기준 실제 사이트
├── firebase.json → dist 를 가리킴 (Vite 앱 산출물)
└── deploy/       nginx.conf · DEPLOY_GUIDE.md
```

`handoff.md`에 "main 브랜치: Next.js 15 마이그레이션 완료"라고 적혀 있고 현재 `next/`는 16이다. 루트 Vite 앱이 마이그레이션 이전 잔재라면 통째로 제거하는 게 가장 큰 정리가 된다.

**임의로 지우지 않는다.** 어느 쪽이 실제 서비스되는지 확인이 먼저다 — 배포 워크플로와 nginx 설정이 무엇을 가리키는지 본다.

---

## 하지 않을 것

- **my-ui-lib 1.0.0 설치** — 쓰는 곳이 없으므로 의미가 없다
- **profile 툴체인 최신화** — 루트 Vite 앱의 TS 5.7 · Vite 6 · ESLint 9는 my-ui-lib과 같은 상황이지만, 3번에서 그 앱을 지우기로 하면 헛일이 된다. 3번 판단 이후로 미룬다
- **`handoff.md`의 배포 자동화 미완 작업** — 라우터 포트포워딩, SSH 포트 변경, Cloudflare DNS 등. 별건이고 물리적 작업이 섞여 있다

## 순서

```
3 (판단)  →  2 (정리)  →  1 (내용 갱신)
```

3번 결론이 2번의 대상 자체를 없앨 수 있으므로 판단이 먼저다. 1번은 독립적이라 언제든 가능하고, 급하면 1번만 먼저 해도 된다.

## 미해결

- 루트 Vite 앱 생존 여부 — 배포 워크플로 확인 필요
- `next/`에 my-ui-lib 컴포넌트를 실제로 쓸 계획이 있는지 (있다면 `^1.0.0` 설치가 의미를 갖는다)
