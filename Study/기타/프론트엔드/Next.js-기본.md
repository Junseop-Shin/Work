# Next.js 기본 — 라우팅 / 렌더링 / 캐싱 / Edge / SEO

> 2026-05-06 | Next.js, App Router, Rendering, Caching, Edge Runtime, SEO

## 한 줄 요약

Next.js는 React 위에 **라우팅 · 빌드 시점 분리(SSG/SSR/ISR/RSC) · 이미지·폰트 최적화 · 서버 캐싱 · Edge 런타임 · SEO** 를 표준화한 풀스택 프레임워크다. 본질은 "React만으로 안 되는 프로덕션 영역을 컨벤션으로 묶어 둔 것".

## 핵심 개념

### 왜 Next.js를 쓰는가

React 단독의 한계:

- **라우팅 없음** — react-router 직접 붙여야 함
- **번들 큼** — 모든 JS를 클라이언트에서 받음 → SEO/초기 로딩 불리
- **이미지/폰트 최적화 직접** — webpack loader 직접 설정
- **데이터 페칭 컨벤션 없음** — useEffect로 클라이언트 로딩
- **SSR 직접 구현 어려움** — Express + ReactDOMServer 직접 작성

Next가 채워주는 것:

- File-based routing
- 빌드 타임/런타임 자동 분리 (SSG/SSR/ISR/RSC)
- `next/image`, `next/font` 등 표준 최적화
- 풀스택 (API + UI 한 프로젝트)
- Vercel과 함께 배포 표준

비교 대상: Vite + React Router (가벼움, SPA), Remix/React Router v7 (data-loader 중심), Astro (콘텐츠 사이트), TanStack Start (full-stack with TanStack).

### Pages Router → App Router 변천사

원래 Pages Router 였고, **Next 13(2022.10)** 에서 App Router 도입.

| 버전 | 시점 | App Router 상태 |
|------|------|----------------|
| Next 1~12 | 2016~2022 | Pages Router only |
| Next 13 | 2022.10 | App Router beta |
| Next 13.4 | 2023.05 | App Router stable |
| Next 14 | 2023.10 | App Router 추천 |
| Next 15 | 2024.10 | App Router 메인스트림 |
| Next 16 | 2025.10 | App Router 표준 |

**둘 다 한 프로젝트에서 공존 가능.** `app/`과 `pages/` 동시에 두면 점진 마이그레이션. 같은 URL이면 `app/`이 우선.

App Router가 만들어진 이유:

1. Pages는 데이터 페칭이 page 최상단에 묶임 → 자식 컴포넌트가 직접 fetch 못 함 (워터폴)
2. 컴포넌트 단위 SSR/SSG 불가 — 페이지 단위로만 결정
3. 레이아웃 공유 어색함 (`_app.tsx` 한 개)
4. Streaming/RSC 지원 어려움
5. 모든 컴포넌트가 클라이언트 번들로 → 번들 큼

App Router 해결: RSC 기반 + 폴더 단위 layout/loading/error + 컴포넌트 단위 async fetch + Streaming + Server Actions.

---

# 1. 라우팅 (App Router 중심)

## 특수 파일 (폴더 안에 두는 것들)

| 파일명 | 역할 | 동작 |
|--------|------|------|
| `page.tsx` | 페이지 UI | 이 파일 있어야 라우트로 노출 |
| `layout.tsx` | 레이아웃 | children 받아 감쌈. navigation 시 **재마운트 X** (state 유지) |
| `template.tsx` | 템플릿 | layout과 동일 위치인데 navigation 시 **매번 재마운트** |
| `loading.tsx` | Suspense fallback | 해당 segment 로딩 중 표시 (Suspense 자동 감쌈) |
| `error.tsx` | Error Boundary | segment 에러 catch. `reset()` prop으로 재시도 |
| `global-error.tsx` | 루트 에러 바운더리 | layout.tsx가 throw해도 잡힘. `<html>` 직접 그려야 함 |
| `not-found.tsx` | 404 UI | `notFound()` 호출 시 표시 |
| `route.ts` | API 엔드포인트 | GET/POST/PUT/DELETE export. `page.tsx`와 같은 경로 공존 불가 |
| `default.tsx` | Parallel route 기본 fallback | 슬롯 매칭 안 될 때 표시 |
| `middleware.ts` | 요청 미들웨어 | 프로젝트 루트 위치, 라우트 폴더 안 아님 |

## 경로 컨벤션 (폴더명에 의미 부여)

### `[slug]` — Dynamic Segment
```
app/posts/[slug]/page.tsx → /posts/abc, /posts/xyz
```
`page.tsx`에서 `params: { slug: 'abc' }` (Next 15부터 async).

### `[...slug]` — Catch-all
```
app/docs/[...slug]/page.tsx
→ /docs/a, /docs/a/b, /docs/a/b/c 모두 매칭
→ params: { slug: ['a', 'b', 'c'] }
```

### `[[...slug]]` — Optional Catch-all
```
app/shop/[[...slug]]/page.tsx
→ /shop, /shop/a, /shop/a/b 모두 매칭 (루트 포함)
```

### `(group)` — Route Group (URL에서 무시)
```
app/(marketing)/about/page.tsx → /about (괄호는 URL에 안 붙음)
app/(marketing)/layout.tsx     → /about, /home에 공통 적용
app/(shop)/cart/page.tsx       → /cart (다른 layout)
```
같은 URL 깊이에서 다른 layout 그룹을 만들거나, URL 영향 없이 폴더 정리할 때.

### `_folder` — Private Folder (라우팅 제외)
```
app/_components/Button.tsx → 라우트 아님
```
밑줄 prefix는 컴포넌트/유틸 폴더용.

### `@slot` — Parallel Route Slot
```
app/dashboard/
  layout.tsx
  page.tsx
  @analytics/page.tsx
  @team/page.tsx
```
layout이 두 슬롯을 props로 받음:
```jsx
export default function Layout({ children, analytics, team }) {
  return (
    <>
      {children}
      <aside>{analytics}</aside>
      <aside>{team}</aside>
    </>
  );
}
```
한 페이지 안에서 여러 RSC 트리를 독립 렌더. 각 슬롯 자체 loading/error 가능.

### `(.)`, `(..)`, `(...)` — Intercepting Routes
- `(.)` — 같은 레벨 가로채기
- `(..)` — 한 단계 위 가로채기
- `(..)(..)` — 두 단계 위
- `(...)` — root부터 가로채기

대표 사용처: **Instagram-style 모달** (피드에서 사진 클릭 → 모달, URL 새로고침 시 풀 페이지).

### 일반 문자/하이픈
```
app/blog-posts/page.tsx → /blog-posts (그냥 폴더명)
```
하이픈은 일반 문자. URL 슬러그에 권장되는 케밥-케이스.

## `api/` 폴더는 특별한가?

**App Router에선 컨벤션일 뿐 magic 아님.** `route.ts`만 있으면 어디서든 API.

```
app/api/users/route.ts → GET /api/users (관례)
app/products/route.ts  → GET /products (똑같이 동작, 단 page.tsx와 공존 불가)
```

관례상 `app/api/` 아래 모으는 게 page와 충돌 방지에 안전.

(Pages Router는 다름 — `pages/api/*`는 진짜 특수 폴더로 처리됨.)

---

# 2. 렌더링 모드

## CSR (Client-Side Rendering)

**흐름:** 서버는 빈 HTML + JS 번들 → 브라우저가 JS 실행 → DOM 생성 → useEffect로 fetch.

- **장점**: 인터랙션 풍부, SPA 스타일
- **단점**: SEO 불리 (크롤러가 빈 HTML만 봄), 초기 화면 느림, Core Web Vitals 나쁨
- **언제**: 로그인 후 대시보드, 사내 어드민, SEO 무관한 앱
- **Next에서**: `"use client"` + `useEffect` fetch

## SSR (Server-Side Rendering)

**흐름:** 매 요청마다 서버에서 React 실행 → 완성 HTML → 브라우저에 즉시 표시 → JS hydrate.

- **장점**: SEO 우수, 빠른 첫 화면 (FCP), 요청별 fresh 데이터
- **단점**: 서버 부하, TTFB 느릴 수 있음, 인프라 복잡도
- **언제**: 사용자별 대시보드, 자주 바뀌는 콘텐츠, 검색 결과
- **Next에서**: App Router 기본. Server Component에서 fetch 데이터에 cache 옵션 안 주면 SSR (Next 15부터 fetch 기본 no-cache)

## SSG (Static Site Generation)

**흐름:** 빌드 타임에 서버에서 모든 페이지 미리 렌더 → HTML 파일 → CDN 업로드 → 사용자 요청 시 정적 파일 반환.

- **장점**: 가장 빠름, 무한 확장, 저렴, SEO 최강
- **단점**: 데이터 stale, 빌드 시간 폭증 (페이지 100만 개면 빌드도 100만 번), 인터랙티브 데이터 한계
- **언제**: 블로그, 문서, 마케팅 페이지
- **Next에서**: Server Component가 dynamic API(`cookies()`, `headers()`, async params, no-store fetch) 안 쓰면 자동 SSG

`generateStaticParams`로 dynamic 라우트의 SSG 대상 결정:
```ts
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map(p => ({ slug: p.slug }));
}
```

## ISR (Incremental Static Regeneration)

**SSG + 자동 재생성.** SSG의 stale 문제 보완.

**흐름:**
1. 빌드 타임에 SSG 생성
2. 요청 시 캐시된 정적 파일 반환 (즉시)
3. 동시에 백그라운드에서 재생성 → 캐시 갱신
4. 다음 사용자는 새 페이지

**시간 기반:**
```ts
export const revalidate = 60; // 60초마다 재검증
```
첫 요청 후 60초 동안 캐시 반환. 60초 지난 첫 요청은 stale 페이지 반환 + 백그라운드 재생성 (**stale-while-revalidate**).

**On-demand:**
```ts
import { revalidatePath, revalidateTag } from 'next/cache';
revalidatePath('/posts/[slug]', 'page');
revalidateTag('posts');
```
CMS에서 글 발행 시 webhook으로 호출 → 즉시 갱신.

- **언제**: e-커머스 상품, 뉴스, "거의 안 바뀌지만 가끔 바뀜"

## RSC (React Server Components)

**다른 축의 개념.** SSR/SSG는 "언제 렌더하나", RSC는 "컴포넌트가 어디서 실행되나".

- Server Component — 서버에서만 실행, JS 번들에 포함 X
- Client Component — 브라우저에서 hydrate, 번들에 포함 O

한 페이지가 SSR될 수도 SSG될 수도 있고, 그 안에 Server/Client가 섞여 있을 수 있음. 상세는 React 18+19 문서 참조.

## Streaming

페이지를 **chunk 단위로 흘려보냄.** Suspense 경계마다 fallback 먼저, 데이터 준비된 부분 추가 전송.

```jsx
<>
  <Header />
  <Suspense fallback={<Skeleton />}>
    <SlowComments />  {/* 이 부분만 늦게 도착 */}
  </Suspense>
</>
```

효과: TTFB ↓, LCP에 유리, UX 좋음 (스켈레톤 → 실제 콘텐츠 점진 등장).

## PPR (Partial Prerendering)

**한 페이지 안에 정적 부분 + 동적 부분 혼합.** Next 15 experimental, 16 안정화 예상.

```jsx
export const experimental_ppr = true;

export default function Page() {
  return (
    <>
      <StaticHeader />              {/* SSG로 prerender */}
      <Suspense fallback={<Skel />}>
        <DynamicCart />             {/* 요청 시 fetch */}
      </Suspense>
    </>
  );
}
```
빌드 타임에 정적 shell 생성 → 사용자 요청 시 shell 즉시 전송 + 동적 hole만 streaming. **SSG 속도 + SSR 신선함.**

---

# 3. 이미지/폰트 최적화

## `next/image`

```jsx
import Image from 'next/image';
<Image src="/photo.jpg" alt="..." width={800} height={600} />
```

자동 적용:
- **Lazy loading** — viewport 진입 시 로드
- **srcset 자동 생성** — 디바이스별 적정 사이즈
- **포맷 협상** — WebP/AVIF 지원 브라우저엔 자동 변환
- **CLS 방지** — width/height로 공간 미리 확보
- **blur placeholder** — `placeholder="blur"`

핵심 함정: width/height 또는 fill 필수. fill 쓰면 부모에 `position: relative` 필요.

## `next/font`

```jsx
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'] });

<html className={inter.className}>...</html>
```

- 빌드 타임에 Google Fonts 다운로드해서 self-host (CDN 요청 0)
- `font-display: swap` 자동
- CSS 변수로 사용 가능 (`variable: '--font-inter'`)

---

# 4. 데이터 페칭 + API

## Server Component 직접 fetch

```jsx
async function Page() {
  const posts = await fetch('https://api/posts').then(r => r.json());
  return <List items={posts} />;
}
```
async 컴포넌트 자체가 가능. 클라이언트 hook 없이 데이터 가져옴.

## Route Handlers (`app/api/route.ts`)

```ts
export async function GET(req: Request) {
  return Response.json({ ok: true });
}
export async function POST(req: Request) { ... }
```
Pages Router의 `pages/api/*` 대체.

## Server Actions

React 19 섹션에서 다룸. 폼 submit / 버튼 클릭 → 서버 함수 직접 호출.

---

# 5. 미들웨어 + 메타데이터

## Middleware

```ts
// middleware.ts (프로젝트 루트)
export function middleware(req: NextRequest) {
  if (!req.cookies.get('auth')) {
    return NextResponse.redirect(new URL('/login', req.url));
  }
}
export const config = { matcher: ['/dashboard/:path*'] };
```
모든 요청 가로채서 인증/리다이렉트/A-B 테스트/지역 분기 등. **Edge runtime에서 실행** (빠름, 단 Node API 제한).

## Metadata API

```ts
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { images: [post.image] },
  };
}
```
react-helmet 같은 라이브러리 불필요.

---

# 6. 캐싱 4레이어 (가장 헷갈리는 영역)

Next의 캐싱은 **4개 레이어가 협력**. 요청 하나가 어떻게 흘러가는지가 핵심.

## 4가지 캐시

| 레이어 | 위치 | 범위 | 수명 | 무효화 |
|-------|-----|------|------|--------|
| **① Request Memoization** | 서버 메모리 | 한 React 렌더 트리 안 | 렌더 끝나면 소멸 | 자동 (불가 옵션) |
| **② Data Cache** | 서버 디스크 | 모든 사용자 공유 | 영구 (revalidate 시까지) | `revalidatePath` / `revalidateTag` / 시간 |
| **③ Full Route Cache** | 서버 디스크 | 모든 사용자 공유 | 영구 | `revalidatePath` / 시간 / 빌드 |
| **④ Router Cache** | 브라우저 메모리 | 사용자 1명 (탭) | 페이지 닫으면 소멸 | `router.refresh()` / 시간 (Next 15부터 0) |

## 요청 흐름 — 사용자가 `/posts/abc` 클릭했을 때

```
[브라우저]
  사용자가 <Link href="/posts/abc"> 클릭
       ↓
[④ Router Cache 확인]  ← 클라이언트 메모리
  hit? → 즉시 표시 (서버 요청 0)
  miss? → 서버로 요청
       ↓
[③ Full Route Cache 확인]  ← 페이지가 정적이거나 ISR된 경우
  hit? → 캐시된 HTML+RSC payload 그대로 반환
  miss? → 페이지 렌더 시작
       ↓
[Server Components 실행]
  async function Page() {
    const post = await fetch('/api/posts/abc');
    const author = await fetch('/api/users/' + post.authorId);
  }
       ↓
[① Request Memoization]  ← React 트리 내 dedupe
  같은 렌더 트리 안에서 같은 URL+옵션 fetch가 이미 있나?
  yes → 그 결과 재사용
  no  → 다음 단계
       ↓
[② Data Cache 확인]  ← 서버 디스크
  fetch에 cache: 'force-cache' 또는 next.revalidate 옵션 있나?
  yes + hit → 캐시된 결과 반환
  yes + miss → Origin fetch → Data Cache에 저장
  no (Next 15 기본) → Origin fetch (캐시 안 함)
       ↓
[Origin (DB / 외부 API)]
       ↓
[렌더 결과 생성]
       ↓
[③ Full Route Cache에 저장]  ← 정적 페이지면
       ↓
[브라우저로 응답]
       ↓
[④ Router Cache에 저장]
       ↓
[화면에 표시]
```

## 각 레이어 깊게

### ① Request Memoization (React 기능)

**한 렌더 트리 안에서만** 동작. 같은 fetch가 여러 컴포넌트에 흩어져도 한 번만.

```jsx
async function Page({ params }) {
  const post = await fetch(`/api/posts/${params.slug}`);
  return <Article post={post} />;
}

async function Sidebar({ params }) {
  const post = await fetch(`/api/posts/${params.slug}`);  // 같은 URL
  return <Meta post={post} />;
}
```
→ Page와 Sidebar 같이 렌더되면 fetch는 **1번만**. React 자동 dedupe.

→ prop drilling 안 해도 되고, 필요한 곳에서 직접 fetch해도 안전. 트리 끝나면 메모리에서 사라짐.

### ② Data Cache (Next 서버)

`fetch()` 결과를 **서버 디스크에 영구 저장** — 다음 요청, 다른 사용자도 재사용.

```ts
// 캐시 활성화 (Next 15부터 명시 필요)
const res = await fetch('/api/posts', { cache: 'force-cache' });

// 60초 ISR
const res = await fetch('/api/posts', { next: { revalidate: 60 } });

// 태그 부여
const res = await fetch('/api/posts', { next: { tags: ['posts'] } });

// 캐시 안 함 (Next 15 기본)
const res = await fetch('/api/posts', { cache: 'no-store' });
```

**무효화:** 시간 (`revalidate: 60`), on-demand (`revalidatePath('/posts')` 또는 `revalidateTag('posts')`).

`unstable_cache`(15) / `cache()`(16)는 임의 함수를 Data Cache에 넣는 API:
```ts
const getUser = unstable_cache(
  async (id) => db.user.findUnique({ where: { id } }),
  ['user'],
  { revalidate: 60, tags: ['user'] }
);
```

### ③ Full Route Cache (Next 서버)

**완성된 페이지 통째로 (HTML + RSC payload) 디스크에 저장.**

자동 결정 규칙:
- Server Component만 사용 + dynamic API 미사용 → **자동 SSG, Full Route Cache O**
- 위 중 하나라도 사용 → **SSR, Full Route Cache X**

명시적 제어:
```ts
export const dynamic = 'force-static';   // 강제 SSG
export const dynamic = 'force-dynamic';  // 강제 SSR
export const revalidate = 60;            // ISR
```

**무효화:** `revalidatePath()`, `revalidate` 시간 경과, 빌드 재실행.

### ④ Router Cache (클라이언트)

브라우저 메모리에 **RSC payload** 저장. `<Link>` prefetch도 여기 들어감.

- 뒤로가기 — 즉시 표시 (서버 0)
- 앞으로가기 — 즉시 표시
- `<Link>` 호버 — prefetch
- `router.push()` — 캐시 hit 시 즉시 navigation

**Next 15 변화** — default `staleTime: 0` (매번 stale 취급). 이전(Next 14)엔 30초.

```js
// next.config.js
experimental: {
  staleTimes: { dynamic: 30, static: 180 },
},
```

**무효화:** `router.refresh()`, `revalidatePath()`(서버 캐시 무효화 시 클라이언트도 자동), 시간 경과.

## Next Data Cache vs TanStack Query

자주 헷갈리는 비교:

| | **TanStack Query (TQ)** | **Next Data Cache** |
|---|---|---|
| 위치 | 브라우저 메모리 | 서버 디스크 |
| 범위 | 사용자 1명, 탭 1개 | **모든 사용자 공유** |
| 수명 | 페이지 닫으면 사라짐 | **영구 (서버 재시작해도 유지)** |
| 무효화 | `queryClient.invalidateQueries()` | `revalidatePath()` / `revalidateTag()` |
| 누가 fetch | 클라이언트가 직접 | 서버가 RSC 안에서 fetch |

**역할 분담:**
- **정적/반정적 데이터** (글 목록, 상품 카탈로그) → Next Data Cache (모든 사용자 공유, 빠름)
- **사용자 인터랙티브 데이터** (실시간 댓글, mutation 후 갱신, polling) → TQ
- **mutation** → Server Actions

Next Data Cache는 "**서버 측 TanStack Query**"라고 보면 됨. 단 모든 사용자가 공유하므로 더 강력.

## 실전 예시

### 블로그 글 (정적, 가끔 수정)
```ts
export const revalidate = 3600;  // 1시간마다 재생성

export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map(p => ({ slug: p.slug }));
}

async function Page({ params }) {
  const post = await fetch(`https://cms/${params.slug}`, {
    next: { tags: ['post-' + params.slug] },
  });
  return <Article post={post} />;
}
```
→ Full Route Cache O (1시간 ISR), Data Cache O (태그로 on-demand 무효화). CMS 편집 시 webhook으로 `revalidateTag('post-' + slug)`.

### 사용자 대시보드 (개인화, 항상 최신)
```ts
export const dynamic = 'force-dynamic';

async function Page() {
  const user = await getCurrentUser();  // cookies() 사용
  const data = await fetch('/api/dashboard', { cache: 'no-store' });
  return <Dashboard user={user} data={data} />;
}
```
→ Full Route Cache X, Data Cache X. 매 요청 fresh.

---

# 7. Edge Runtime

## 두 가지 런타임

| | **Node.js Runtime** | **Edge Runtime** |
|---|---|---|
| 기반 | Node.js 풀 환경 | V8 isolate (가벼운 V8 인스턴스) |
| 위치 | 단일 리전 서버 | 전 세계 CDN edge 노드 |
| 콜드 스타트 | 수백 ms ~ 초 | **ms 단위** |
| 메모리 | GB 가능 | MB 단위 (~128MB) |
| API | 모든 Node + npm | Web 표준 API + 일부 |
| DB 연결 | TCP 풀링 OK | TCP 어려움 → HTTP API 권장 |

## "Edge"가 뭐냐

**Edge = CDN 노드.** 전 세계 곳곳의 서버 (Cloudflare/Vercel/Fastly가 운영).

- 기존 SSR: 서울 사용자 → 미국 서버 → 다시 서울 (왕복 200ms+)
- Edge SSR: 서울 사용자 → 도쿄 edge 노드 → 서울 (왕복 30ms)

→ **사용자와 가까운 곳에서 코드 실행** = 지연 ↓

## V8 isolate가 뭐냐

- **Node.js**: V8 엔진 + Node 런타임(libuv, fs, net) 통째로 띄움. 무거움
- **Edge**: V8 엔진만 가볍게 띄움. 한 프로세스 안에 수만 개의 isolate 운영 가능. 시작 시간 ms

→ **콜드 스타트 거의 없음** (서버리스의 가장 큰 단점 해결)

## 일반 SSR vs Edge SSR 그림

### 일반 SSR (PM2/Docker로 직접 운영)
```
[서울 사용자] ──► CDN(도쿄, 정적 자산만) ──► 메인 서버(미국 EC2, PM2) ──► DB
```
HTML 받으려면 매번 미국까지 왕복. CDN은 정적 파일 가속기 역할만.

### Edge SSR (Vercel/Cloudflare 플랫폼)
```
[서울 사용자] ──► Edge 노드(도쿄) ──► DB(HTTP 기반)
                  - 정적 자산
                  - SSR 코드 실행 ★
                  - 미들웨어 실행
```
사용자와 가까운 도쿄에서 SSR. 메인 서버가 사라지거나 부수 역할만.

## 미들웨어는 어디서 실행되나

- **Vercel/Cloudflare 플랫폼**: 무조건 Edge — 전 세계 모든 edge 노드에서 실행
- **Self-host (PM2/Docker)**: 그냥 Node 메인 서버에서 실행. 지리적 분산 효과 X

## "Edge runtime" 명시의 진짜 의미

```ts
export const runtime = 'edge';
```

이 한 줄이 의미하는 두 가지:

1. **코드 환경 제약** — Node 전용 API 못 씀, Web 표준만 사용 가능 (어디서 실행되든)
2. **(Vercel/Cloudflare 같은 플랫폼 배포 시) edge 노드에서 실행**

Self-host 환경에선 1번만 적용되고 2번 효과 X. 그래서 self-host 프로젝트에선 `runtime = 'nodejs'`(기본)로 두고 Edge runtime 거의 안 씀.

## 누가 Edge 노드를 운영하나

- **Cloudflare Workers** — 200+ 도시 자체 데이터센터
- **Vercel Edge Functions** — Cloudflare Workers 인프라 위에 빌드
- **AWS Lambda@Edge** — CloudFront 위
- **Deno Deploy** — Deno 자체 인프라

우리는 코드만 올림 → 그들이 200+ 도시에 자동 복제 배포 → 사용자 가장 가까운 노드에서 실행.

## Web 표준 API만 사용

쓸 수 있는 것: `fetch()`, `Request`, `Response`, `URL`, `Headers`, `crypto.subtle` (Web Crypto), `TextEncoder`, `TextDecoder`, `ReadableStream`, `setTimeout`, `setInterval`, `console`, `URLPattern` ✅

**못 쓰는 것**: `fs`, `child_process`, `net`/`dgram`(raw TCP/UDP), `process.cpuUsage`/`os.platform()` 같은 OS API, Node 전용 npm 패키지 다수, Native modules ❌

## 제약

- **번들 크기**: Cloudflare Workers 1MB, Vercel Edge 4MB
- **CPU 시간**: 보통 50~수백 ms
- **메모리**: 보통 128MB
- **TCP 연결 풀링 어려움** → DB 직접 연결 X. 대안:
  - HTTP API 기반 DB (Neon HTTP, Supabase REST, PlanetScale HTTP)
  - Upstash Redis (HTTP 기반)
  - Drizzle + HTTP driver

## 언제 Edge를 쓰나

| 적합 ✅ | 부적합 ❌ |
|--------|----------|
| 인증 토큰 검증 | 무거운 데이터 처리 |
| A/B 테스트 분기 | 복잡한 트랜잭션 DB 쿼리 |
| Geo 기반 콘텐츠 분기 | 이미지/파일 처리 (sharp 등 native) |
| Bot 차단 / Rate limiting | 머신러닝 추론 |
| 간단한 API (KV 조회) | 큰 의존성 라이브러리 |
| Streaming SSR (TTFB ↓) | Node 전용 npm |
| Middleware (인증/리다이렉트) | 긴 CPU 작업 |

## 실전 예시

### Geo 분기 미들웨어
```ts
// middleware.ts
import { NextResponse } from 'next/server';

export function middleware(req) {
  const country = req.geo?.country;
  if (country === 'KR') {
    return NextResponse.rewrite(new URL('/ko', req.url));
  }
  return NextResponse.next();
}
```

### 인증 토큰 검증 미들웨어
```ts
export async function middleware(req) {
  const token = req.cookies.get('auth')?.value;
  if (!token) return NextResponse.redirect(new URL('/login', req.url));
  const valid = await verifyJWT(token);  // Web Crypto
  if (!valid) return NextResponse.redirect(new URL('/login', req.url));
}

export const config = { matcher: ['/dashboard/:path*'] };
```

### Edge에서 DB 쿼리 (HTTP 기반 Neon)
```ts
import { neon } from '@neondatabase/serverless';

export const runtime = 'edge';
const sql = neon(process.env.DATABASE_URL!);

export async function GET() {
  const rows = await sql`SELECT * FROM posts LIMIT 10`;
  return Response.json(rows);
}
```

---

# 8. SEO

## 검색엔진의 한계와 Next의 해결

### 1) HTML이 비어 있으면 인덱싱 못 함
- CSR: 봇이 빈 div만 봄 → 인덱싱 0 또는 매우 늦음
- **Next 해결**: SSR/SSG로 완성 HTML 전송

### 2) 메타데이터 (검색 결과 표시 정보)

검색 결과의 제목/설명, SNS 공유 카드 등.

```ts
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.coverImage, width: 1200, height: 630 }],
      type: 'article',
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      images: [post.coverImage],
    },
    alternates: {
      canonical: `https://mysite.com/posts/${post.slug}`,
    },
  };
}
```
`alternates.canonical` — 같은 콘텐츠가 여러 URL에 있을 때 정식 주소 지정. 중복 콘텐츠 페널티 방지.

### 3) sitemap.xml

```ts
// app/sitemap.ts
import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getAllPosts();
  return [
    { url: 'https://mysite.com', lastModified: new Date(), priority: 1 },
    ...posts.map(p => ({
      url: `https://mysite.com/posts/${p.slug}`,
      lastModified: p.updatedAt,
      changeFrequency: 'weekly' as const,
    })),
  ];
}
```
파일 만들면 `/sitemap.xml`로 자동 노출.

### 4) robots.txt

```ts
// app/robots.ts
export default function robots() {
  return {
    rules: [{ userAgent: '*', allow: '/', disallow: ['/admin/', '/api/'] }],
    sitemap: 'https://mysite.com/sitemap.xml',
  };
}
```

### 5) 구조화 데이터 (JSON-LD)

검색 결과에 별점, 가격, FAQ 같은 rich snippet:
```jsx
export default function Page() {
  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: post.title,
    author: { '@type': 'Person', name: post.author },
    datePublished: post.publishedAt,
  };
  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <Article post={post} />
    </>
  );
}
```

### 6) Core Web Vitals (Google 랭킹 신호)

2021년부터 Google이 검색 순위에 반영. Next 기여:

| 지표 | Next 기여 |
|------|----------|
| **LCP** (Largest Contentful Paint) | `next/image`로 이미지 빨리 로드 + 적정 사이즈, SSR/SSG로 빠른 첫 페인트 |
| **CLS** (Cumulative Layout Shift) | `next/image` width/height 강제, `next/font`로 폰트 로드 시 layout shift 제거 |
| **INP** (Interaction to Next Paint) | RSC로 클라이언트 JS 줄여 메인 스레드 부담 감소 |

`useReportWebVitals`로 측정값 수집 가능:
```jsx
import { useReportWebVitals } from 'next/web-vitals';

useReportWebVitals(metric => {
  analytics.track('web_vital', metric);
});
```

### 7) 다국어 SEO (`hreflang`)

```ts
export const metadata: Metadata = {
  alternates: {
    languages: { 'en-US': '/en', 'ko-KR': '/ko' },
  },
};
```

### 8) Open Graph 이미지 동적 생성

```ts
// app/posts/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og';

export default async function og({ params }) {
  const post = await getPost(params.slug);
  return new ImageResponse(
    <div style={{ ... }}>{post.title}</div>,
    { width: 1200, height: 630 }
  );
}
```
SNS 공유 카드 이미지를 빌드/요청 시점에 자동 생성. 디자이너가 글마다 만들 필요 X.

---

# 9. 환경변수 + 배포

## 환경변수

- `process.env.FOO` — 서버에서만 사용 가능
- `process.env.NEXT_PUBLIC_FOO` — **빌드 타임에 클라이언트 번들로 인라인됨**
- `.env.local` (gitignore), `.env.production`

## 빌드 타임 vs 런타임

- `NEXT_PUBLIC_` 변수는 **빌드 타임에 박힘** → 런타임에 못 바꿈
- 런타임 변경 필요하면 server에서 fetch해서 props로 넘기기

## Standalone output

```js
// next.config.js
output: 'standalone',
```
배포에 필요한 최소 파일만 추출. Docker 이미지 작아짐.

## Edge vs Node runtime

- **Node** (기본) — 모든 npm 패키지 사용 가능
- **Edge** — 가볍고 빠름 (글로벌 분산), 단 일부 Node API 제한
- Route Handler/Page에 `export const runtime = 'edge'`

---

## 핵심 질의응답

**Q. Pages Router → App Router 변천사는?**
A. Next 1~12까지 Pages만, 13(2022.10)에 App Router beta, 13.4(2023.05) stable, 15부터 메인스트림. 한 프로젝트에서 둘 다 공존 가능. 같은 URL이면 `app/`이 우선.

**Q. App Router는 왜 만들어졌나?**
A. Pages는 데이터 페칭이 page 최상단에만 가능 → 워터폴 발생, 컴포넌트 단위 SSR/SSG 불가, 레이아웃 공유 어색, RSC/Streaming 미지원, 번들 큼 — 이런 구조적 한계 해결을 위해 RSC 기반으로 재설계.

**Q. SSR과 RSC의 차이?**
A. SSR은 "언제 HTML을 만들 것인가"(요청 시), RSC는 "컴포넌트가 어디서 실행되는가"(서버에서만). 다른 축의 개념. RSC를 SSR로 렌더할 수도, SSG로 prerender할 수도 있음.

**Q. ISR vs SSG vs SSR 결정 기준?**
A. 데이터 변경 빈도 + 사용자별 차이 여부.
- 거의 안 바뀜, 모든 사용자 동일 → **SSG**
- 가끔 바뀜, 모든 사용자 동일 → **ISR**
- 매번 바뀜 또는 사용자별 다름 → **SSR**

**Q. Next Data Cache가 TanStack Query를 대체하나?**
A. 부분적으로. 정적/반정적 데이터(서버 fetch)는 Next Data Cache가 더 강력 (모든 사용자 공유, 영구). 인터랙티브 데이터(클라이언트 mutation, polling, infinite scroll)는 여전히 TQ가 필요. 역할 분담 권장.

**Q. Edge Runtime을 self-host에서 쓰면?**
A. 코드 환경 제약(Node API 못 씀)만 적용되고, 지리적 분산 효과는 없음. 그냥 메인 서버에서 실행됨. Self-host에선 `runtime = 'nodejs'`(기본) 권장. Vercel/Cloudflare 같은 플랫폼 배포 시에만 진짜 Edge 효과.

**Q. 미들웨어는 어디서 실행되나?**
A. Vercel/Cloudflare 플랫폼: 무조건 Edge (전 세계 노드). Self-host: 메인 서버. 코드는 같지만 실행 환경 다름.

**Q. 캐싱 4레이어 우선순위?**
A. ④ Router Cache (브라우저) → ③ Full Route Cache (서버) → 페이지 렌더 → ① Request Memoization (트리 내) → ② Data Cache (서버 디스크) → Origin. 각 단계마다 hit하면 다음 단계 안 감.

**Q. Next 15부터 fetch가 기본 캐시 안 함이 된 이유?**
A. 의도치 않은 stale 데이터 노출 방지. 이전엔 모든 fetch가 자동 캐시되어 "왜 데이터가 안 바뀌지?" 혼란 빈발. 캐시 원하면 명시적으로 `force-cache` 또는 `revalidate` 옵션 줘야 함.

**Q. SEO를 위해 Next에서 꼭 해야 할 것?**
A. ① SSR/SSG 사용 (빈 HTML 금지) ② `generateMetadata`로 title/description/OG 설정 ③ `app/sitemap.ts`, `app/robots.ts` 작성 ④ `next/image`, `next/font`로 Core Web Vitals 개선 ⑤ canonical URL 지정 ⑥ 구조화 데이터(JSON-LD) 필요 시.

## 주의사항 / 자주 하는 실수

- **`api/` 폴더는 App Router에선 magic 아님** — 그냥 컨벤션. `route.ts` 위치만 중요. 단 page.tsx와 같은 segment에 공존 불가
- **`page.tsx` 없는 폴더는 라우트 아님** — 그냥 `layout.tsx`만 있으면 자식 라우트가 그 layout 사용. URL로 접근 X
- **Server Component에서 dynamic API(`cookies()`, `headers()`, async params, no-store fetch) 사용 시 자동 SSR 전환** — 의도와 다른 동적 렌더 발생 가능. 의도적이면 OK, 아니면 그 부분만 별도 컴포넌트로 격리
- **`"use client"`는 모듈 그래프 경계** — 그 컴포넌트가 import한 자식들도 모두 client
- **Edge Runtime에서 Node API 호출하면 빌드 에러** — `fs`, `child_process` 등 사용 시 자동 감지
- **`NEXT_PUBLIC_` 변수는 빌드 타임에 박혀서 못 바꿈** — 런타임 변경 필요하면 서버에서 fetch
- **Next 15부터 fetch 기본 no-cache** — 14에서 마이그레이션 시 의도치 않은 캐시 미스로 DB 부하 ↑ 가능. `force-cache` 또는 `revalidate` 명시 점검
- **Router Cache `staleTime: 0` (Next 15)** — navigation 시 매번 서버 요청. 14에서 30초였던 동작과 다름
- **Mongoose Document, class instance를 RSC props로 직접 넘기면 직렬화 에러** — `.toObject()` 또는 plain mapping 거쳐야 함

## 참고

- [Next.js 공식 문서](https://nextjs.org/docs)
- [App Router 마이그레이션 가이드](https://nextjs.org/docs/app/guides/migrating/app-router-migration)
- [Caching 레이어 공식 설명](https://nextjs.org/docs/app/deep-dive/caching)
- [Edge Runtime API Reference](https://nextjs.org/docs/app/api-reference/edge)
- [Metadata API](https://nextjs.org/docs/app/api-reference/functions/generate-metadata)
- [버전별 변경사항은 별도 문서](Next.js-15-16-주요변경사항.md)
