# Next.js 15 + 16 주요 변경사항

> 2026-05-06 | Next.js, App Router, Async Request APIs, Turbopack, Cache Components

## 한 줄 요약

Next 15는 **fetch/route 캐시 기본값을 뒤집고**(no-cache가 기본) **request API를 async로 통일**해 RSC 시대에 맞는 명시적 모델로 전환했고, Next 16은 **Turbopack을 build까지 기본으로** 끌어올리고 **React Compiler를 자동 활성화**해 도구 체인을 새로 잠궜다.

## 핵심 개념

기본 / 캐싱 / 렌더링 / 라우팅 등 일반론은 [Next.js 기본 문서](Next.js-기본.md) 참조. 이 문서는 **15와 16에서 새로 들어왔거나 동작이 바뀐 항목**만 정리.

---

# Part 1. Next.js 15 (2024-10 출시)

## 1. Async Request APIs (Breaking)

가장 큰 변화. `params`, `searchParams`, `cookies()`, `headers()`, `draftMode()`이 **모두 async**.

```ts
// Before (Next 14)
function Page({ params }: { params: { slug: string } }) {
  const { slug } = params;
}

// Next 15
async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
}
```

```ts
// cookies / headers
import { cookies, headers } from 'next/headers';

const cookieStore = await cookies();
const headerList = await headers();
```

### 왜?

이 값들이 진짜로 사용될 때까지 React가 렌더를 진행할 수 있게 하기 위함. Async로 만들어 두면 PPR(Partial Prerendering) 같은 시나리오에서 정적 부분은 미리 렌더하고 dynamic 부분만 늦게 처리 가능.

Next 15는 sync 사용도 deprecation warning과 함께 임시 허용. **Next 16부터 sync 사용 시 에러.**

codemod 제공:
```bash
npx @next/codemod@latest next-async-request-api
```

## 2. Caching 기본값 변경 (Breaking)

가장 큰 동작 변화. 캐시 layer별로 기본값 뒤집힘.

| 항목 | Next 14 | Next 15 |
|------|---------|---------|
| `fetch()` GET | **자동 캐시** (`force-cache`) | **캐시 X** (`no-store`) |
| GET Route Handler (`route.ts`) | 자동 캐시 | 캐시 X |
| Client Router Cache | `staleTime: 30s` | **`staleTime: 0`** |

### 영향

이전엔 의도치 않게 stale 데이터 노출되는 사례 빈발 → "왜 데이터 안 바뀌어?" 혼란. Next 15는 **opt-in 모델**로 전환:

```ts
// 캐시 원하면 명시
const res = await fetch('/api/posts', { cache: 'force-cache' });
const res = await fetch('/api/posts', { next: { revalidate: 60 } });
```

### 마이그레이션 주의

기존 코드가 자동 캐시에 의존하던 부분에서 **DB 부하 급증** 가능. fetch 호출부 점검 필요. 거의 안 바뀌는 데이터엔 명시적 `force-cache` 추가.

### Router Cache 조절

`staleTime: 0`이 너무 공격적이면:
```js
// next.config.js
experimental: {
  staleTimes: { dynamic: 30, static: 180 },
},
```

## 3. React 19 정식 지원

- App Router는 React 19 (RC → stable)
- Pages Router는 React 18/19 둘 다 지원

→ Server Actions, `useActionState`, `useFormStatus`, `useOptimistic`, `use()`, React Compiler 사용 가능. 상세는 React 18+19 문서 참조.

## 4. Turbopack Dev 정식

`next dev --turbo` → `next dev` 기본. webpack은 대안.

내부: Rust 기반 incremental compilation. webpack 대비 cold start ~76% 빠름, 변경 후 파일 갱신 ~96% 빠름 (공식 벤치).

**단 build는 아직 webpack** (Next 16부터 build도 Turbopack 기본).

## 5. `<Form>` 컴포넌트 (`next/form`)

HTML form에 client-side navigation 자동 적용:
```jsx
import Form from 'next/form';

<Form action="/search">
  <input name="query" />
  <button>Search</button>
</Form>
```
- JS 비활성 시 정상 form submit (progressive enhancement)
- JS 활성 시 prefetch + soft navigation
- search 파라미터 자동 처리

## 6. `unstable_after` API

Response 보낸 후 백그라운드 작업 실행 (로깅, 분석 등).

```ts
import { unstable_after as after } from 'next/server';

export default function Page() {
  after(() => {
    log('analytics event');  // 응답 후 백그라운드 실행
  });
  return <div>...</div>;
}
```

응답 지연 없이 부수 작업 처리. Vercel은 함수 종료 후에도 일정 시간 컴퓨팅 보장.

## 7. `instrumentation.js` 정식화

서버 시작 시 1회 실행. OpenTelemetry, Sentry 등 초기화에 사용. 더 이상 experimental 아님.

```ts
// instrumentation.ts (프로젝트 루트)
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./instrumentation-node');
  }
}
```

## 8. Static Route Indicator

개발 모드에서 페이지가 static인지 dynamic인지 화면 좌하단에 표시. 의도치 않게 SSR로 떨어지는 페이지 즉시 발견.

## 9. Self-hosting 개선

- 이미지 최적화 시 `sharp` 자동 사용 (별도 설치 불필요)
- Cache-Control 헤더 더 세밀하게 제어
- 환경 변수 처리 개선

## 10. `next.config.ts` 지원

config 파일을 TS로 작성 가능:
```ts
// next.config.ts
import type { NextConfig } from 'next';

const config: NextConfig = {
  experimental: { reactCompiler: true },
};

export default config;
```

## 11. ESLint 9 지원

flat config (`eslint.config.js`) 지원.

## 12. create-next-app 개선

- 새 design (CLI UI)
- Turbopack 기본 옵션
- src/ 구조 옵션 명확

---

# Part 2. Next.js 16 (2025-10 출시 추정)

## 13. Turbopack 기본 (build 포함)

`next build`도 Turbopack 사용. webpack 옵션 남아있지만 deprecated 방향.

영향:
- 빌드 시간 대폭 단축
- 일부 webpack-specific 플러그인 호환성 이슈 가능 → 마이그레이션 필요

## 14. React Compiler 기본 활성화

`reactCompiler: false` 명시 안 하면 켜진 상태.

→ `useMemo`/`useCallback` 거의 불필요. 단 React Rules 위반 코드는 컴파일러가 자동 skip.

준비:
```bash
npm install -D eslint-plugin-react-hooks@latest
```
lint 통과부터 시작 권장 (상세는 React 18+19 문서 참조).

## 15. Cache Components

`unstable_cache`를 대체. RSC에서 사용자 정의 캐시 컴포넌트 작성 가능.

```ts
// 개념적 사용
import { cache } from 'react';
import { unstable_cacheLife } from 'next/cache';

const getUser = cache(async (id: string) => {
  unstable_cacheLife({ revalidate: 60, tags: [`user-${id}`] });
  return db.user.findUnique({ where: { id } });
});
```

→ 함수 내부에서 캐시 정책 선언. fetch 외 임의 데이터 소스에도 캐시 적용.

## 16. Async params 강제 (이중 호환 종료)

Next 15는 sync도 deprecated 경고로 허용. **16부터 sync 사용 시 에러.**

마이그레이션 안 한 코드는 16 업그레이드 전 codemod 실행:
```bash
npx @next/codemod@latest next-async-request-api
```

## 17. Node.js 20.9+ 최소

구버전 Node 지원 종료. CI/배포 환경 확인 필요.

## 18. Next DevTools (브라우저 통합)

React DevTools 같은 Next 전용 디버깅 도구. 캐시 상태, RSC payload, Server Action 추적 가능.

## 19. Middleware 개선

- Edge runtime에서 Node API 일부 추가 (Web Crypto 확장 등)
- matcher 성능 향상
- middleware chain 개념 강화 (여러 middleware 순차 실행)

## 20. Image 최적화 변화

- `<Image>`의 `sizes` 추론 개선
- AVIF 기본 지원
- 더 똑똑한 포맷 협상

---

## 핵심 질의응답

**Q. Next 15에서 fetch가 기본 no-cache로 바뀐 이유?**
A. 의도치 않은 stale 데이터 노출 방지. 이전엔 모든 fetch가 자동 캐시되어 "왜 데이터가 안 바뀌지?" 혼란 빈발. 캐시 원하면 명시적으로 `force-cache` 또는 `revalidate` 옵션 줘야 함.

**Q. Async params로 바뀐 진짜 이유?**
A. PPR(Partial Prerendering) 같은 시나리오에서 정적 부분 먼저 렌더하고 dynamic 값(`params`, `cookies` 등)은 나중에 처리하기 위함. async 아니면 dynamic 값이 필요한 시점에 렌더 전체가 멈춤. async로 바꾸면 React가 적절히 일정 잡아 처리.

**Q. Turbopack이 webpack보다 빠른 이유?**
A. ① **Rust 기반** (JS보다 빠름) ② **incremental compilation** (변경된 부분만 재컴파일, 캐시 재사용) ③ **함수 단위 캐싱** (모듈보다 더 fine-grained) ④ **lazy compilation** (실제 import된 코드만 컴파일).

**Q. React Compiler를 16에서 기본 켜는데 위험은?**
A. React Rules 위반 코드(state mutation, render 중 ref 변경 등)에서 잘못된 메모이제이션이 들어가면 stale UI 버그 발생. 단, 컴파일러는 lint 통과 못 한 컴포넌트는 자동 skip하므로 lint 우선 통과 시키면 안전. 16 전에 `eslint-plugin-react-hooks` 통과부터.

**Q. Cache Components가 unstable_cache와 다른 점?**
A. ① 함수 정의 내부에서 수명/태그 선언 (선언적) ② RSC와 더 자연스럽게 통합 ③ fetch가 아닌 임의 데이터 소스(Redis, file, 외부 API) 모두 적용 가능. unstable_cache는 wrapper 함수로 감싸야 했음.

**Q. `<Form>` 컴포넌트와 일반 `<form action={serverAction}>`의 차이?**
A. `<Form>`은 GET 기반 navigation에 최적화 (검색, 필터). server action 호출이 아니라 URL 이동. `<form action={serverAction}>`은 mutation용. `<Form>`은 prefetch도 자동.

**Q. unstable_after의 실제 활용?**
A. 응답 보낸 후 비동기 작업: 분석 이벤트 전송, 로그 기록, 캐시 워밍, 외부 시스템 동기화. 사용자 응답 지연 없이 부수 작업 처리. Vercel/Cloudflare 함수에서 종료 후 컴퓨팅 보장.

**Q. instrumentation.js는 언제 쓰나?**
A. 서버 startup 시 1회 실행이 필요한 모든 것: OpenTelemetry/Sentry 초기화, DB 풀 warmup, 외부 서비스 health check, 백그라운드 워커 spawn 등.

## 마이그레이션 체크리스트

### Next 14 → 15

- [ ] `npx @next/codemod@latest upgrade` 실행
- [ ] Async Request API codemod (`next-async-request-api`) 실행
- [ ] `params`, `searchParams`, `cookies()`, `headers()` 모두 `await` 처리 확인
- [ ] **fetch 호출 점검** — 캐시 의존하던 부분에 명시적 `force-cache` / `revalidate` 추가
- [ ] **GET Route Handler 점검** — 캐시 의존 시 명시 옵션
- [ ] Router Cache `staleTime: 0` 영향 확인 → 필요 시 `staleTimes` 옵션 설정
- [ ] React 19 마이그레이션 (forwardRef → ref prop, deprecated API 점검)
- [ ] Turbopack 시도 (`next dev --turbo`)
- [ ] ESLint 9 flat config로 전환 (선택)

### Next 15 → 16

- [ ] Node.js 20.9+ 환경 확인
- [ ] Async API codemod 재실행 (sync 잔여 제거)
- [ ] React Compiler 활성화 전 `eslint-plugin-react-hooks` 통과
- [ ] 빌드를 Turbopack으로 전환 시 webpack-specific 플러그인 호환성 확인
- [ ] `unstable_cache` → Cache Components 마이그레이션 검토
- [ ] middleware chain 동작 변화 확인

## 주의사항 / 자주 하는 실수

- **15 마이그레이션 시 fetch 캐시 미스로 DB 부하 ↑** — 명시적 캐시 옵션 점검
- **15에서 sync params/cookies 쓰면 deprecation warning만**, 16에선 에러 — 미리 codemod 실행
- **`<Form>`을 mutation에 쓰지 말 것** — GET 전용. mutation은 `<form action={serverAction}>` 또는 `useActionState`
- **`unstable_after`는 응답 후 실행이라 사용자에게 결과 못 줌** — 사용자에게 결과 보여줘야 하는 작업엔 부적합
- **React Compiler 활성화 후 stale UI 버그가 보이면** lint 위반부터 점검 (mutation, render 중 부수효과 등)
- **Turbopack 빌드는 일부 webpack 플러그인 미지원** — 16 업그레이드 전 빌드 환경 확인

## 참고

- [Next.js 15 공식 announcement](https://nextjs.org/blog/next-15)
- [Next.js 16 공식 announcement](https://nextjs.org/blog/next-16)
- [Async Request API 마이그레이션](https://nextjs.org/docs/app/guides/upgrading/version-15)
- [Caching 레이어 공식 설명](https://nextjs.org/docs/app/deep-dive/caching)
- [Turbopack 문서](https://nextjs.org/docs/app/api-reference/turbopack)
- [기본 개념은 별도 문서](Next.js-기본.md)
