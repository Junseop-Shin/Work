# React 18 + 19 주요 변경사항

> 2026-05-06 | React, Concurrent Rendering, Server Components, React Compiler, Server Actions

## 한 줄 요약

React 18은 **렌더링을 인터럽트 가능하게** 만들어 동시성(Concurrency)의 토대를 깔았고, React 19는 **Server-First 아키텍처(RSC/Server Actions)** 와 **컴파일러 기반 자동 메모이제이션** 으로 React를 다시 한 번 패러다임 단위로 바꿨다.

## 핵심 개념

- **React 18**: Concurrent Rendering 인프라 도입. 단, 이건 **opt-in**이라 `useTransition`/`useDeferredValue`/`<Suspense>` 같은 API를 명시적으로 써야 활성화됨.
- **React 19**: Server Components / Server Actions / Compiler — 사람이 손으로 하던 메모이제이션과 RPC 보일러플레이트를 프레임워크 레이어로 흡수.

---

# Part 1. React 18 — Concurrent Foundation

## 1. Concurrent Rendering

**핵심**: 렌더링이 "**인터럽트 가능**"해졌다.

### 17까지의 한계

렌더링은 한 번 시작하면 끝까지 동기적으로 실행. 중간에 멈출 수 없음. 큰 트리를 렌더링할 때 메인 스레드를 점유해서 사용자 입력이 끊기는 jank가 발생.

### 18의 변화

렌더링(reconciliation)과 커밋(DOM 반영)을 분리:

- **렌더링**: 중단 가능, 우선순위 비교, 버려지기도 함(thrown away)
- **커밋**: 여전히 동기적, 한 번에 DOM 반영

**우선순위 기반 렌더링**: 저우선순위 렌더링(예: 검색 결과 리스트) 진행 중에 사용자가 새 키 입력 같은 고우선순위 업데이트를 발생시키면, React가 **진행 중이던 저우선순위 작업을 버리고** 고우선순위 먼저 처리. 그 후 다시 저우선순위 재시작.

### 함정 ⚠️

**Concurrent Rendering은 자동이 아니다. Opt-in이다.**

React 18로 업그레이드만 한다고 동시성이 켜지지 않음. 두 가지가 필요:

1. `ReactDOM.createRoot()` 사용 (기존 `ReactDOM.render`는 legacy 모드)
2. `useTransition`, `useDeferredValue`, `<Suspense>` 같은 동시성 API를 명시적으로 사용

기존 코드는 18로 올려도 그냥 동기적으로 동작.

### 받쳐주는 아키텍처

**Fiber** (React 16 도입). Fiber는 가상 DOM 노드를 **연결 리스트**로 표현해서 렌더링을 chunk 단위로 쪼개고, 각 chunk 사이에 브라우저에 제어권 양보(yield)할 수 있게 만든 자료구조. Concurrent Rendering은 Fiber가 있었기 때문에 가능.

## 2. Automatic Batching

**핵심**: 여러 setState를 한 번의 리렌더로 묶는다.

### 17까지

React 이벤트 핸들러 안에서만 batching:

```jsx
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // → 1번 리렌더 ✅
}

fetch('/api').then(() => {
  setCount(c => c + 1); // 리렌더 1
  setFlag(f => !f);     // 리렌더 2 ❌
});
```

### 18부터

**모든 곳에서 batching**. async 콜백, setTimeout, native 이벤트 핸들러까지.

명시적 opt-out: `flushSync`

```jsx
import { flushSync } from 'react-dom';

flushSync(() => {
  setCount(c => c + 1); // 즉시 DOM 반영
});
const node = ref.current; // 새 DOM 측정 가능
setFlag(f => !f);
```

언제 필요한가: setState 직후 DOM을 측정해야 할 때 (리스트 추가 후 스크롤 위치 계산 등), 외부 라이브러리와 동기화할 때.

### 마이그레이션 함정

기존 코드가 "여러 번 리렌더되는 중간 상태"에 의존하면 깨질 수 있음. 대부분은 오히려 정합성이 좋아지지만, race condition 형태로 우연히 동작하던 코드가 드러남.

## 3. `useTransition` / `useDeferredValue`

Concurrent를 활용하는 실제 API.

### `useTransition`

"이 업데이트는 긴급하지 않다"고 표시.

```jsx
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setSearchQuery(input); // 저우선순위로 처리
});
```

→ 사용자 입력은 즉시 반영, 무거운 리스트 필터링은 백그라운드. `isPending`으로 로딩 UI 가능.

### `useDeferredValue`

값 자체를 "지연된 버전"으로 만든다.

```jsx
const deferredQuery = useDeferredValue(query);
// query는 즉시 업데이트, deferredQuery는 한 박자 늦게
```

### 둘의 차이

- `useTransition`: **state 업데이트를** 지연. 내가 setState하는 위치에 적용
- `useDeferredValue`: **값을** 지연. 외부에서 props로 받는 값(내가 setState 못 하는 값)에 적용

## 4. Suspense for Data Fetching + Streaming SSR

### Suspense 확장

React 16에선 `lazy()` 코드 스플리팅 전용. **18부터 데이터 페칭에도 사용 가능**.

```jsx
<Suspense fallback={<Spinner />}>
  <Comments /> {/* 안에서 throw하는 promise를 Suspense가 catch */}
</Suspense>
```

### Streaming SSR

- `renderToString` (동기, 한 번에) → **`renderToPipeableStream`** (스트림)
- 서버가 HTML을 **chunk 단위로 흘려보냄**. Suspense 경계마다 fallback 먼저 보내고, 데이터 준비되면 그 부분만 추가 전송
- **Selective Hydration**: 클라이언트가 받은 부분부터 즉시 hydrate, 사용자가 클릭한 부분 우선 hydrate

이게 RSC와 Streaming의 토대.

## 5. `useId`

서버/클라이언트에서 동일한 고유 ID 생성. SSR에서 hydration mismatch 방지용.

```jsx
const id = useId();
<input id={id} />
<label htmlFor={id}>...</label>
```

## 6. Strict Mode 변화

개발 모드에서 effect를 **mount → unmount → re-mount** 두 번 실행. cleanup 제대로 작성했는지 검증. 미래의 "state 보존하며 재사용" 기능(Offscreen 등)을 대비.

---

# Part 2. React 19 — Server-First & Compiler 🎯

## 7. React Compiler

**가장 큰 패러다임 변화.** `useMemo`, `useCallback`, `React.memo`를 수동으로 안 써도 됨.

### 왜 필요한가 (배경)

React는 state가 바뀌면 컴포넌트를 **다시 실행**. 그러면 함수 안의 모든 변수, `.map()`, 객체 리터럴이 매 렌더마다 새로 만들어짐.

```jsx
function List({ items }) {
  const sorted = items.slice().sort();    // 매 렌더 새 배열
  const handleClick = () => {...};         // 매 렌더 새 함수
  return <Child onClick={handleClick} data={sorted} />;
}
```

`Child`가 `React.memo`여도, `handleClick`/`sorted`가 매번 새 참조 → props "바뀐 걸로" 인식 → 매번 리렌더. → 사람이 일일이 `useMemo`/`useCallback` + 의존성 배열 관리. 빠뜨리면 stale closure 버그.

**컴파일러가 이걸 자동화.**

### 작동 원리 (4단계, 풀어서)

#### ① HIR (High-level Intermediate Representation) 생성

코드를 컴파일러가 분석하기 쉬운 "중간 형태"로 변환. 마치 영어 문장을 분석하기 전 "주어/동사/목적어"로 라벨링하듯, 각 연산을 작은 단위로 쪼개고 입력-출력을 명확히 표시.

```jsx
const sorted = items.slice().sort();

// 컴파일러 내부 표현 (개념)
$1 = items
$2 = $1.slice()       // 입력: $1
$3 = $2.sort()        // 입력: $2
sorted = $3
```

> SSA-like — 각 변수에 한 번만 값 할당해서 추적 쉽게.

#### ② 의존성 추적

각 값이 어디서 왔는지 그래프 작성.

```
items (props)
  ↓
sorted = items.slice().sort()
  ↓
<Child data={sorted} />
```

→ "**`sorted`는 `items`에만 의존**. `items` 안 바뀌면 `sorted` 재계산 불필요." 사람이 손으로 적던 `useMemo(() => ..., [items])`의 `[items]`를 컴파일러가 자동 추출.

#### ③ 재실행 단위 분석

"어디까지 묶어서 캐시할까?" 결정.

```jsx
function Page({ user, posts }) {
  const greeting = `Hello, ${user.name}`;        // user에만 의존
  const sorted = posts.slice().sort();            // posts에만 의존
  const summary = `${greeting} - ${posts.length}`; // 둘 다 의존
}
```

→ 각각 별도 캐시 슬롯에 저장. `user`만 바뀌면 `sorted`는 그대로, `greeting`/`summary`만 재계산.

#### ④ 메모이제이션 캐시 코드 삽입

분석 결과를 실제 JS로 변환. `useMemoCache(N)`이라는 React 내부 훅으로 N개 슬롯 배열을 받아 입력값/결과값 저장.

```jsx
// 작성 코드
function List({ items }) {
  const sorted = items.slice().sort();
  return <ul>{sorted.map(i => <li>{i}</li>)}</ul>;
}

// 컴파일 후 (개념)
function List({ items }) {
  const $ = useMemoCache(2);
  let sorted;
  if ($[0] !== items) {
    sorted = items.slice().sort();
    $[0] = items;
    $[1] = sorted;
  } else {
    sorted = $[1];  // 캐시 재사용
  }
  return <ul>{sorted.map(i => <li>{i}</li>)}</ul>;
}
```

`useMemo` 수동 작성과 동등. 단, **모든 파생 값에 자동 적용**.

### 적용 방법

#### 1) 사전 lint 검증 (필수)

React Rules 위반 시 컴파일러가 잘못된 메모를 생성. ESLint로 먼저 차단:

```bash
npm install -D eslint-plugin-react-hooks@latest
```

```js
// eslint.config.js
import reactHooks from 'eslint-plugin-react-hooks';
export default [reactHooks.configs['recommended-latest']];
```

lint 통과 못 하는 파일은 컴파일러가 자동 skip. **lint 에러 다 잡고 시작.**

#### 2) 컴파일러 설치

```bash
npm install -D babel-plugin-react-compiler@latest
```

#### 3) Next.js 활성화

```js
// next.config.js
module.exports = {
  experimental: {
    reactCompiler: true,
    // 또는 reactCompiler: { compilationMode: 'annotation' }
  },
};
```

#### 4) Vite/Babel 직접

```js
// babel.config.js
module.exports = {
  plugins: [['babel-plugin-react-compiler', { compilationMode: 'infer' }]],
};
```

### `compilationMode` 3가지

| 모드 | 동작 | 언제 |
|------|------|------|
| `annotation` | `"use memo"` 디렉티브 붙은 함수만 컴파일 | 점진 도입 초기 |
| `infer` | 컴포넌트로 추정한 함수만 (대문자 시작 + JSX 반환) | **권장 기본값** |
| `all` | 모든 함수 컴파일 시도 | 검증 끝난 성숙한 코드베이스 |

### 점진 도입 전략

- **Phase 1 — Annotation**: leaf 컴포넌트에 `"use memo"` 붙여 시작 → 검증 후 위로 확장
- **Phase 2 — Infer + opt-out**: 거의 다 자동, 문제 컴포넌트만 `"use no memo"`로 제외
- **Phase 3 — 디렉티브 제거**: 충분히 검증되면 모두 떼고 100% 위임

### 함정

- **즉시 적용 안 됨** — React Rules 위반(state mutation, render 중 ref read/write, props mutation)이 많으면 lint 통과시켜야 의미 있음
- **수동 메모 그대로 둬도 됨** — 한꺼번에 제거 불필요. 컴파일러가 더 잘 메모하기도
- **디버깅** — 컴파일 결과물 읽기 어려움. React DevTools에 "Memo ✨" 배지로 표시
- **번들 사이즈** — 약간 증가 (캐시 슬롯 코드). 런타임 이득이 훨씬 큼
- **Next.js 16 기본 활성화 (예정)** — 16부터는 `reactCompiler: false` 명시 안 하면 켜진 상태. 미리 lint 통과시켜 두면 마이그레이션 부드러움

## 8. Server Components (RSC)

**서버에서만 실행되고, JS 번들에 포함되지 않는 컴포넌트.**

### 핵심

- Next.js App Router에서 기본이 Server Component
- `"use client"` 디렉티브 없으면 서버 컴포넌트
- 서버에서 렌더링 결과를 직렬화된 형태(RSC payload)로 클라이언트에 전송
- DB/파일시스템 직접 접근 가능, async 가능
- 브라우저 API/이벤트 핸들러/state/effect 사용 불가 → 그건 Client Component

### 멘탈 모델

```
[Server Component] → JSON-like payload (RSC payload) → [Client]
                                                        ↓
                               [Client Component]는 이 안에서 hydrate
```

Server Component는 **클라이언트 번들에 0바이트** 추가. 큰 마크다운 파서, 코드 하이라이터 같은 라이브러리 마음껏 써도 됨.

### 직렬화 경계 (가장 큰 함정)

**직렬화(serialize)** = 메모리에 있는 객체를 문자열/바이트로 변환해 전송 가능한 형태로 만드는 것.

RSC에서 "Server Component → Client Component"로 props 넘어가는 지점이 **직렬화 경계**. 이 지점에선 JSON으로 변환 가능한 것만 통과 가능.

| ✅ 가능 | ❌ 불가능 |
|---------|----------|
| 원시값 (string, number, boolean, null, undefined) | 일반 함수 (Server Action 제외) |
| 배열, 객체 (plain) | class instance |
| Date, BigInt, Symbol.for(...) registered | DOM 노드 |
| Map, Set | 클라이언트 객체 캡처한 클로저 |
| **Promise**, TypedArray, ArrayBuffer | |
| **Server Action 함수** (`"use server"`) | |
| React 엘리먼트 (RSC payload로 인코딩) | |

#### 함정 1: 함수 전달

```jsx
// ❌ 함수 직렬화 불가
function ServerList() {
  return <ClientItem onClick={() => doSomething()} />;
}

// ✅ Server Action으로 만들거나, 핸들러를 Client Component 안에서 정의
"use server";
async function handleClick() { ... }
<ClientItem onClickAction={handleClick} />
```

#### 함정 2: 클래스 인스턴스

```jsx
// ❌
class User { constructor(name) { this.name = name; } }
return <ClientCard user={new User('A')} />;

// ✅ plain object
return <ClientCard user={{ name: 'A' }} />;
```

Prisma는 plain object 반환해서 안전. **Mongoose Document는 `.toObject()` 필요.**

#### 함정 3: Promise를 props로 (이건 OK, 워터폴 제거)

```jsx
// Server Component
function Page() {
  const dataPromise = fetchData(); // await 안 함
  return (
    <Suspense fallback={<Spinner />}>
      <ClientChart dataPromise={dataPromise} />
    </Suspense>
  );
}

// Client
"use client";
import { use } from 'react';
function ClientChart({ dataPromise }) {
  const data = use(dataPromise); // 여기서 Suspense
  return <Chart data={data} />;
}
```

서버는 await 안 하고 즉시 렌더 시작 → HTML 스트리밍, 클라이언트가 hydrate 후 데이터 받아 렌더. **워터폴 제거.**

#### 함정 4: `"use client"`의 진짜 의미

"여기서부터 클라이언트 경계"라는 뜻. import한 자식들도 자동 client 처리.

```jsx
// ClientParent.tsx
"use client";
import ChildA from './ChildA'; // ChildA는 use client 없어도 client
```

근데 children으로 넘기면 다름:

```jsx
// ServerPage.tsx (Server Component)
return <ClientParent><ServerChild /></ClientParent>;
// ServerChild는 여전히 server
```

→ **children 패턴**으로 server 컴포넌트를 client 안에 끼워넣기. RSC 컴포지션의 핵심 패턴.

#### 함정 5: 서버 모듈 누출 방지

```js
// db.ts
import 'server-only'; // 클라이언트 import 시 빌드 에러
import { Pool } from 'pg';
export const db = new Pool(...);

// 반대: import 'client-only';
```

## 9. Server Actions (`"use server"`)

**클라이언트에서 호출하지만 서버에서 실행되는 함수.**

```jsx
// app/actions.ts
"use server";

export async function createPost(formData: FormData) {
  const title = formData.get('title');
  await db.post.create({ data: { title } });
  revalidatePath('/posts');
}

// app/page.tsx (Client Component에서)
"use client";
import { createPost } from './actions';

<form action={createPost}>
  <input name="title" />
  <button>Submit</button>
</form>
```

### 핵심

- 함수가 **자동으로 RPC 엔드포인트**가 됨 (Next가 내부적으로 POST 라우트 생성)
- `<form action={...}>`에 직접 넘기면 progressive enhancement (JS 꺼져도 동작)
- 보안: 함수 본문이 클라이언트 번들에 안 들어감. 하지만 누구나 호출 가능 → **함수 안에서 인증/권한 체크 필수**

### 보안 패턴

#### 위협 모델

Server Action은 **누구나 호출 가능한 RPC 엔드포인트**. 클라이언트는 함수 ID와 인자만 알면 호출 가능.

```jsx
"use server";
export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });
}
```

→ 공격자가 다른 사람 post id 알아내 호출하면 삭제됨. **인증/권한 체크가 함수 안에 없으면 전부 뚫림.**

#### 패턴 1: 인증 체크 항상 first

```ts
"use server";
export async function deletePost(id: string) {
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');

  const post = await db.post.findUnique({ where: { id } });
  if (post.authorId !== session.user.id) throw new Error('Forbidden');

  await db.post.delete({ where: { id } });
  revalidatePath('/posts');
}
```

#### 패턴 2: 입력 검증 (Zod)

```ts
"use server";
import { z } from 'zod';
const schema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().max(10000),
});

export async function createPost(prevState, formData: FormData) {
  const parsed = schema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });
  if (!parsed.success) return { error: parsed.error.flatten() };
  // ...
}
```

`FormData`는 string이지만 사용자가 임의 데이터 보낼 수 있음. **무조건 검증.**

#### 패턴 3: CSRF 보호

**CSRF (Cross-Site Request Forgery)** = 사용자가 로그인된 상태를 악용해 다른 사이트가 사용자 몰래 요청을 보내는 공격.

시나리오:

1. 사용자가 `mybank.com` 로그인 → 브라우저에 세션 쿠키 저장
2. 사용자가 `evil.com` 방문
3. `evil.com`이 몰래 `mybank.com/transfer`로 form submit
4. 브라우저가 `mybank.com` 도메인 쿠키 자동 첨부 → 서버는 정상 요청으로 인식 → 송금 실행

방어: Next.js는 자동으로 **Origin 헤더 체크**로 CSRF 방어. 도메인 추가 시 명시:

```js
// next.config.js
experimental: {
  serverActions: { allowedOrigins: ['my-app.com', 'preview.my-app.com'] },
},
```

#### 패턴 4: Rate Limiting

같은 사용자/IP가 일정 시간 내 보낼 수 있는 요청 수 제한. 예방하는 공격:

- **브루트포스**: 비밀번호 자동 시도 (1분 5회 제한 → 1억 후보 시도 수년 걸림)
- **Credential Stuffing**: 유출된 계정 정보로 대량 로그인 시도
- **Account Enumeration**: "이 이메일 가입돼 있냐?" 무한 조회로 회원 명단 수집
- **댓글/메시지 스팸**: 봇이 광고 무한 등록
- **리소스 고갈**: 비싼 연산(이미지/AI) 무한 호출로 서버 다운
- **비용 폭탄**: LLM API 같은 종량 과금 폭파

```ts
"use server";
import { ratelimit } from '@/lib/ratelimit';

export async function postComment(text: string) {
  const session = await auth();
  const { success } = await ratelimit.limit(session.user.id);
  if (!success) throw new Error('Too many requests');
  // ...
}
```

#### 패턴 5: Server Action ID 안정성

빌드 시 Server Action에 **암호화된 ID** 자동 부여. 멀티 인스턴스 배포 시 동일 키 필수:

```js
// next.config.js
serverActions: { encryptionKey: process.env.SERVER_ACTIONS_KEY }, // 32바이트 base64
```

키 다르면 인스턴스 간 action ID 불일치로 `Action invocation failed`.

#### 절대 하지 말 것

- ❌ 클라이언트가 보낸 user id를 그대로 신뢰 (`{ userId: formData.get('userId') }`) → **항상 session에서**
- ❌ Server Action을 inline 정의해서 sensitive 데이터 클로저 캡처 → 클로저 변수도 클라이언트가 조작 가능
- ❌ Action 결과를 throw로만 처리 → 에러 메시지에 내부 정보 포함되면 정보 유출. 구조화된 `{ error }` 객체 반환 권장

## 10. Actions 패러다임 — `useActionState`, `useFormStatus`, `useOptimistic`

### `useActionState` (구 `useFormState`)

폼 액션의 결과/pending 상태를 관리.

```jsx
const [state, formAction, isPending] = useActionState(createPost, initialState);

<form action={formAction}>
  <input name="title" />
  {state?.error && <p>{state.error}</p>}
  <button disabled={isPending}>Submit</button>
</form>
```

### `useFormStatus`

**부모 form의 상태**를 자식 컴포넌트가 읽음. 버튼 컴포넌트를 따로 분리해도 form 상태 접근 가능.

```jsx
function SubmitButton() {
  const { pending } = useFormStatus(); // 가장 가까운 form 상태
  return <button disabled={pending}>Submit</button>;
}
```

### `useOptimistic` — 풀어서

#### 문제 상황

사용자가 "좋아요" 클릭:

```jsx
async function handleLike() {
  await api.like(postId);  // 서버 호출 (200ms)
  setLikes(likes + 1);     // 응답 후 UI 업데이트
}
```

→ 200ms 동안 아무 반응 없음. "어, 안 눌렸나?" 하고 또 누름. 답답.

#### 해결: 미리 UI 업데이트 (수동)

```jsx
async function handleLike() {
  setLikes(likes + 1);          // 먼저 UI 업데이트
  try {
    await api.like(postId);
  } catch (e) {
    setLikes(likes);            // 실패 시 되돌리기
    showError();
  }
}
```

직접 짜려니 문제: 실패 시 롤백 코드 매번, 빠르게 여러 번 클릭 시 state 꼬임, 서버 응답이 다른 값 줄 때 동기화 어려움.

#### `useOptimistic`이 하는 일

**"진짜 state"와 별개로 "낙관적 임시 state"를 자동 관리.**

```jsx
const [likes, setLikes] = useState(10);  // 진짜 state
const [optimisticLikes, addOptimistic] = useOptimistic(
  likes,                                  // 진짜 값 참조
  (currentLikes, increment) => currentLikes + increment  // 어떻게 미리 바꿀지
);

async function handleLike() {
  addOptimistic(1);              // ① "+1 미리 보여줘"
  await api.like(postId);        // ② 서버 호출
  setLikes(prev => prev + 1);    // ③ 진짜 값 갱신
}
```

화면엔 `optimisticLikes`를 표시.

| 시점 | likes (진짜) | optimisticLikes (화면) |
|------|----------|----------|
| 클릭 전 | 10 | 10 |
| `addOptimistic(1)` 직후 | 10 | **11** ← 즉시 반영 |
| `await api.like` 진행 중 | 10 | 11 |
| `setLikes(11)` 후 | 11 | 11 (자동 추적) |

핵심:

- 서버 응답 후 진짜 값(`setLikes`) 업데이트하면 → optimistic이 자동으로 진짜 값으로 reset
- **에러 시 수동 롤백 불필요** — transition 종료 → optimistic 자동 폐기 → realState로 화면 복귀

#### 왜 transition 안에서만 동작?

`useOptimistic`은 **transition 컨텍스트에 묶임**. transition 진행 중에만 유효, 끝나면 자동 폐기.

이게 안 그러면: 빠르게 여러 번 submit 시 optimistic state 누적되어 꼬임, 에러 발생 시 어떻게 되돌릴지 애매. transition을 lifecycle로 활용 → 시작 시 optimistic 적용, 성공/실패 시 자동 reset.

#### 함정

- **transition 밖 호출 시 에러** — `"An optimistic state update occurred outside a transition"`. Server Action은 자동 transition 안에서 실행되어 안전. 직접 호출 시 `startTransition`으로 감싸야 함
- **realState 안 바꾸면 optimistic이 영원히 안 사라짐** — transition 종료 시 realState로 되돌아가는데, realState 그대로면 optimistic이 화면에서 사라져 보임. 서버 응답 후 **반드시 setState 또는 `revalidatePath`**
- **RSC 환경 권장 패턴** — `revalidatePath`와 조합:

```jsx
"use server";
export async function addTodo(text: string) {
  await db.todo.create({ data: { text } });
  revalidatePath('/'); // RSC payload 갱신 → todos prop 새 값 → optimistic 자동 reset
}
```

## 11. `use()` 훅

**Promise나 Context를 조건부로 읽기.** 다른 훅과 달리 if/loop 안에서도 사용 가능.

### Promise 읽기

```jsx
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise); // resolve까지 Suspense
  return <ul>{comments.map(...)}</ul>;
}

<Suspense fallback={<Spinner />}>
  <Comments commentsPromise={fetchComments()} />
</Suspense>
```

부모에서 Promise를 만들어 자식에게 넘기면, 자식이 `use()`로 받아서 Suspense가 처리. 부모는 await 안 하고 즉시 렌더 시작 → **워터폴 방지**.

### Context 읽기

```jsx
function Component() {
  if (condition) {
    const theme = use(ThemeContext); // 조건부 가능 ✅
  }
}
```

`useContext`는 조건부 호출 불가, `use`는 가능.

## 12. `ref` as a prop, `forwardRef` 폐기

```jsx
// React 18
const Input = forwardRef((props, ref) => <input ref={ref} {...props} />);

// React 19
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}
```

`ref`가 일반 prop처럼 동작. 기존 `forwardRef` 코드는 deprecation warning만 뜨고 동작은 유지.

## 13. Document Metadata 자동 hoisting

`<title>`, `<meta>`, `<link>`를 컴포넌트 안에서 작성하면 React가 자동으로 `<head>`로 끌어올림.

```jsx
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title}</title> {/* 자동으로 <head>로 */}
      <meta name="description" content={post.excerpt} />
      <h1>{post.title}</h1>
    </article>
  );
}
```

react-helmet 같은 라이브러리 불필요. (Next.js는 `metadata` API를 더 권장)

## 14. Stylesheet & Resource Preloading

```jsx
<link rel="stylesheet" href="form.css" precedence="default" />
```

```jsx
import { preinit, preload, prefetchDNS } from 'react-dom';

preinit('https://.../script.js', { as: 'script' });
preload('https://.../font.woff2', { as: 'font' });
prefetchDNS('https://api.example.com');
```

라우트 진입 시점에 미리 리소스 받아놓기 가능.

## 15. Context as Provider

```jsx
// React 18
<MyContext.Provider value={value}>...</MyContext.Provider>

// React 19
<MyContext value={value}>...</MyContext>
```

## 16. Async Transitions

`startTransition`이 **async 함수**를 받음. 전체 async 흐름이 transition으로 처리됨. Actions 시스템의 토대.

```jsx
startTransition(async () => {
  await doSomething();
  setState(newValue);
});
```

## 17. 기타 자잘한 변화

- **에러 처리 개선** — hydration error 메시지가 훨씬 명확
- **`<Context>` displayName** 자동 부여
- **Custom Elements 지원** — web component props/attribute 매핑 정상화
- **`onScrollEnd` 이벤트** 지원

---

## 핵심 질의응답

**Q. Concurrent Rendering이 opt-in인 이유?**
A. 기존 코드와의 호환성 때문. 동기 렌더링에 의존하던 코드가 자동 동시성으로 깨지면 마이그레이션 비용 폭발. 그래서 React는 "동시성 인프라는 깔지만 활성화는 명시적으로" 전략을 선택. createRoot + 동시성 API를 써야 발현.

**Q. Automatic Batching이 깨뜨릴 수 있는 코드 패턴?**
A. setState를 여러 번 호출한 직후 DOM이 즉시 업데이트되리라 가정하던 코드, 또는 setState 사이에 외부 라이브러리/DOM 측정이 끼어 있던 코드. 해결은 `flushSync`로 명시적 분리.

**Q. RSC 직렬화 경계에서 함수를 못 넘기는 이유?**
A. JSON에 함수 표현이 없음. 서버에서 만든 클로저를 클라이언트로 보낼 방법이 없기 때문. Server Action은 예외인데, 함수 본문은 서버에 두고 클라이언트엔 "참조 ID"만 보내서 RPC로 호출하는 구조라 가능.

**Q. CSRF 방어가 Server Action에서 어떻게 되나?**
A. Next.js가 자동으로 요청의 Origin 헤더를 검사해 우리 도메인이 아니면 거부. 별도 CSRF 토큰 코드 불필요. 단, 서브도메인이나 추가 도메인 쓰면 `serverActions.allowedOrigins`에 명시 필요.

**Q. Rate limit이 보안에 좋은 이유?**
A. 자동화된 공격(브루트포스, credential stuffing, account enumeration, 스팸, DoS-lite, LLM 비용 폭탄)을 모두 "시간당 호출 수"로 차단. 일반 API뿐 아니라 Server Action에도 동일하게 필요.

**Q. `useOptimistic`이 transition 안에서만 동작하는 이유?**
A. transition을 optimistic state의 생명 주기로 사용하기 위해. 시작=optimistic 적용, 종료=자동 reset. 이렇게 묶어두지 않으면 여러 번 호출 시 누적/꼬임/롤백 문제가 사용자 코드로 떨어짐.

**Q. React Compiler가 lint를 강요하는 이유?**
A. 컴파일러는 "React Rules가 지켜진다"는 가정 하에 메모이제이션을 자동 생성. 규칙 위반(예: state 직접 mutation, render 중 ref 변경) 코드에 자동 메모를 적용하면 캐시된 잘못된 결과를 재사용해 버그 발생. 그래서 lint 통과 못 한 컴포넌트는 컴파일러가 자동 skip.

**Q. RSC에서 클라이언트 안에 서버 컴포넌트를 어떻게 끼워넣나?**
A. `children` 또는 일반 prop으로 server 컴포넌트를 넘기면 됨. import해서 직접 렌더하면 자동으로 client 컴포넌트로 변환되지만, prop으로 받은 것은 부모(서버)가 이미 렌더한 결과이므로 server 그대로 유지.

## 주의사항 / 자주 하는 실수

- **React 18 업그레이드만으로 동시성이 켜지지 않음** — `createRoot()` + 동시성 API 명시 사용 필요
- **`use client`는 한 파일이 아니라 모듈 그래프 경계** — 그 컴포넌트가 import한 자식들도 모두 client
- **Server Action에 인증 체크 빠뜨리면 즉시 취약점** — public RPC 엔드포인트로 노출됨
- **클라이언트가 보낸 user id를 절대 신뢰하지 말 것** — session에서만 가져오기
- **`useOptimistic`은 transition 밖에서 호출 시 에러** — Server Action 안 쓰면 `startTransition`으로 감싸야 함
- **React Compiler는 lint 위반 코드를 자동 skip** — 의도와 달리 메모이제이션 안 됨. lint 통과부터 시작
- **Mongoose Document, class instance를 RSC props로 직접 넘기지 말 것** — `.toObject()` 또는 plain mapping 거쳐야 함

## 참고

- [React 18 공식 announcement](https://react.dev/blog/2022/03/29/react-v18)
- [React 19 공식 announcement](https://react.dev/blog/2024/12/05/react-19)
- [React Compiler 문서](https://react.dev/learn/react-compiler)
- [Server Components RFC](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)
- [Next.js App Router](https://nextjs.org/docs/app)
