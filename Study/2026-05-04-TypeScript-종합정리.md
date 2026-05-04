# TypeScript 종합 정리 — 기초부터 7.0(Native)까지

> 2026-05-04 | TypeScript, 타입시스템, 제네릭, 유틸리티타입

## 한 줄 요약

TypeScript는 **컴파일 타임 정적 타입 검사기**로, 런타임 안전성은 스키마 검증(zod 등)에 위임하고, 타입 시스템 자체는 자바스크립트의 동적 본질(구조적 타이핑) 위에 정교한 추론·매핑·조건부 연산을 쌓아 올린 별개의 언어다.

## 핵심 개념

### 왜 TypeScript를 쓰는가

진짜 본질은 세 가지:

1. **컴파일 타임 정적 검사** — 런타임 전에 타입 오류를 잡음. 단, 런타임에는 타입이 모두 사라짐(type erasure). `tsc`가 만든 결과물에는 타입 코드가 일절 남지 않음.
2. **DX(개발자 경험)** — 자동완성, 리팩토링 안전성, 타입이 곧 문서. 매일의 생산성은 사실 이쪽이 더 큼.
3. **AI 시대의 정적 검증기** — AI가 hallucinate한 메서드를 빌드 타임에 잡아주고, 타입 시그니처가 AI/사람 모두에게 명확한 명세가 됨.

런타임 안전성에 대한 흔한 착각:

```typescript
const user: User = await res.json(); // ❌ TS는 이걸 검증하지 않고 그냥 믿어버림
```

서버가 잘못된 모양을 보내도 컴파일 통과 → 런타임에 터짐. 그래서 시스템 경계에서는 **zod, valibot, io-ts** 같은 런타임 스키마 검증을 함께 써야 진짜 안전.

```typescript
const UserSchema = z.object({ id: z.number(), name: z.string() });
const user = UserSchema.parse(await res.json()); // 런타임 검증 + 타입 추론
```

### 구조적 타이핑 (Structural Typing)

Java/C#처럼 타입 이름으로 비교하는 명목적(nominal) 타이핑이 아니라, **모양(shape)이 같으면 같은 타입**으로 취급. 덕 타이핑의 정적 버전.

```typescript
interface Duck { quack(): void }
class Dog { quack() {} }
const d: Duck = new Dog(); // ✅ 통과 — 이름이 아니라 모양만 봄
```

장점: 함수가 "필요한 최소 모양"만 명시하면 어떤 타입이든 받음. 단점: 의미적으로 다른데 모양이 같으면 컴파일러가 막을 방법이 없음 (`UserId`와 `ProductId` 둘 다 `string`이면 서로 섞여도 통과).

해결책 — **Branded Types** (nominal 흉내):

```typescript
type UserId = string & { readonly __brand: 'UserId' };
type ProductId = string & { readonly __brand: 'ProductId' };
// __brand는 컴파일 타임에만 존재. 런타임은 그냥 string.
// 하지만 타입 시스템상으론 호환 안 됨.
```

zod의 `.brand<'UserId'>()`로도 동일하게 만들 수 있음.

## 타입 시스템 기초

### `interface` vs `type`

| 항목 | `interface` | `type` |
|------|-------------|--------|
| 객체 모양 정의 | OK | OK |
| Union/Intersection (`\|`, `&`) | ❌ | ✅ |
| Tuple, Mapped, Conditional 등 | ❌ | ✅ |
| **Declaration Merging (선언 병합)** | ✅ | ❌ |
| 성능 (대형 프로젝트) | 약간 유리 (named cache) | intersection 매번 계산 |

가장 본질적인 기능 차이는 **Declaration Merging** — 같은 이름의 `interface`를 여러 번 선언하면 자동 병합. 외부 라이브러리 타입 확장(예: Express `Request`에 `user` 필드 추가)에 필수.

```typescript
declare global {
  namespace Express {
    interface Request { user?: User } // interface만 가능
  }
}
```

**실용 룰**: `interface`로 가능하면 `interface`, 안 되면 `type`.

### `any` / `unknown` / `never`

세 타입의 위치를 한 번에 잡는 그림:

```
        unknown          ← top type (모든 타입의 부모)
          ↑
   string, number, ...   ← 일반 타입
          ↑
        never            ← bottom type (모든 타입의 자식)

  any  ← 이 계층 바깥의 "탈출구" (위험)
```

**`any`** — 타입 시스템을 끄는 탈출구. 어디든 들어가고 어디서든 받음. **전염성**이 있어서 닿은 값이 모두 any가 됨.

```typescript
function process(input: any) {
  return input.map(x => x * 2); // 반환 타입도 any → 호출부 전파
}
```

**`unknown`** — 안전한 any. 어떤 값이든 담을 수 있지만, 좁히기(narrowing) 전엔 아무것도 못 함.

```typescript
const x: unknown = JSON.parse(input);
x.name; // ❌
if (typeof x === 'object' && x !== null && 'name' in x) {
  console.log(x.name); // ✅
}
```

**`never`** — 절대 발생할 수 없는 값의 타입. 함수가 throw하거나 무한 루프일 때, 좁히기 끝까지 가서 남은 게 없을 때 등장.

`never`의 실전 활용 — **Exhaustiveness Check**:

```typescript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; size: number };

function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.radius ** 2;
    case 'square': return s.size ** 2;
    default:
      const _: never = s; // 새 케이스 추가하면 여기서 컴파일 에러 → 처리 누락 자동 감지
      throw new Error(`Unhandled: ${s}`);
  }
}
```

## 제네릭 (Generics)

타입을 매개변수로 받는 타입/함수. `any`와 다른 점은 **타입 정보를 기억해서 반환에 연결**.

```typescript
function identity<T>(value: T): T { return value; }
const x = identity("hello"); // T = "hello" (literal로 추론)
```

### `extends` 제약

`T extends U`는 "T는 최소한 U의 모양을 갖춘 타입이어야 한다". **추가 필드는 그대로 보존**.

```typescript
function first<T extends { id: string }>(arr: T[]): T { return arr[0]; }

const users = [{ id: 'u1', name: 'Alice', age: 30 }];
const u = first(users);
console.log(u.age); // ✅ — name, age도 보존됨
```

### `keyof` + 인덱스 액세스 — 가장 유용한 패턴

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 'u1', name: 'Alice', age: 30 };
const name = getProperty(user, 'name'); // string
const x = getProperty(user, 'foo');     // ❌ 'foo'는 keyof user가 아님
```

### Default Type Parameters

```typescript
function createState<T = string>(initial?: T): T { ... }
```

React의 `useState<T = undefined>()` 가 이런 형태.

### 언제 멈춰야 하는가

| 상황 | 제네릭 |
|------|--------|
| 라이브러리/공용 유틸 | 적극 사용 |
| 애플리케이션 코드 | 단순하게 |
| 3중 이상 중첩 conditional type | 재고 |
| 시그니처가 본문보다 길어짐 | 의심 신호 |

> "타입을 위한 타입은 만들지 말 것" — 항상 "이게 실제 코드를 더 안전하게 만드나?"를 자문.

## 유틸리티 타입

자주 쓰는 빌트인들의 **내부 구현**까지 보면 결국 두 가지 빌딩 블록(Mapped, Conditional+infer)의 조합.

| 유틸 | 효과 | 핵심 구현 |
|------|------|-----------|
| `Partial<T>` | 모든 속성 옵셔널 | `{ [K in keyof T]?: T[K] }` |
| `Required<T>` | 모든 속성 필수 | `{ [K in keyof T]-?: T[K] }` |
| `Readonly<T>` | 모든 속성 readonly | `{ readonly [K in keyof T]: T[K] }` (얕은) |
| `Pick<T, K>` | 키 골라내기 | `{ [P in K]: T[P] }` |
| `Omit<T, K>` | 키 빼기 | `Pick<T, Exclude<keyof T, K>>` |
| `Record<K, V>` | 키-값 매핑 | `{ [P in K]: V }` |
| `ReturnType<T>` | 함수 반환 타입 | `T extends (...a: any) => infer R ? R : any` |
| `Parameters<T>` | 함수 인자 튜플 | `T extends (...a: infer P) => any ? P : never` |
| `Awaited<T>` | Promise 풀어내기 | 재귀적 conditional |
| `NonNullable<T>` | null/undefined 제거 | `T extends null \| undefined ? never : T` |
| `Exclude<T, U>` / `Extract<T, U>` | 유니온 필터 | `T extends U ? never : T` / `T extends U ? T : never` |

### Pick vs Omit — 안전성 차이

`Pick`은 `K extends keyof T`라 오타를 즉시 잡음. `Omit`은 `K extends keyof any`라 잘못된 키도 통과 (사일런트 버그).

```typescript
type User = { id: number; name: string };
type Wrong1 = Pick<User, 'naem'>; // ❌ 즉시 에러
type Wrong2 = Omit<User, 'naem'>; // ✅ 통과 (잘못된 결과)
```

남길 게 적으면 Pick, 뺄 게 적으면 Omit — 가독성과 의도 표현 측면에서 선택.

### `keyof any`

= `string | number | symbol`. JS 객체 키로 가능한 모든 타입의 유니온. `Record<K extends keyof any, V>`에서 K의 제약이 이것.

## 고급 타입 — 빌딩 블록 셋

### 1. Mapped Types

`for...in` 처럼 키를 순회하면서 새 타입 생성. modifier로 `readonly`/`?` 추가(`+`)·제거(`-`).

```typescript
type Variations<T> = { readonly [K in keyof T]?: T[K] };
type Strip<T>      = { -readonly [K in keyof T]-?: T[K] };
```

**Key Remapping (`as` 절, TS 4.1+)** — 키 자체를 변형:

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
// User { id, name } → { getId(): number; getName(): string }
```

`as never`로 매핑된 키는 **결과에서 제거** → 필터링 트릭.

### 2. Template Literal Types

JS 템플릿 리터럴의 타입 레벨 버전. 유니온이 들어가면 자동 분배(데카르트 곱).

```typescript
type Lang = 'en' | 'ko' | 'ja';
type Greeting = `Hello in ${Lang}`;
// = 'Hello in en' | 'Hello in ko' | 'Hello in ja'

type Class = `${'sm'|'md'|'lg'}-${'red'|'blue'}`;
// = 'sm-red' | 'sm-blue' | 'md-red' | ...
```

빌트인 변환: `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize`.

### 3. Conditional Types + `infer`

타입 레벨의 if-else.

```typescript
type IsString<T> = T extends string ? true : false;
```

**`infer`** — 패턴 매칭처럼 일부 타입을 잡아내는 키워드.

```typescript
type ReturnType<T>    = T extends (...a: any) => infer R ? R : never;
type ArrayElement<T>  = T extends (infer E)[] ? E : never;
type PromiseValue<T>  = T extends Promise<infer V> ? V : T;
type FirstElement<T>  = T extends [infer F, ...any[]] ? F : never;
```

**Distributive Conditional Types** — 유니온이 conditional을 만나면 분배:

```typescript
type ToArray<T> = T extends any ? T[] : never;
type X = ToArray<string | number>; // string[] | number[] (← (string|number)[]이 아님)

// 분배 막기 — 대괄호로 감싸기
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type Y = ToArrayNonDist<string | number>; // (string | number)[]
```

`Exclude`가 동작하는 원리이기도 함.

## 실전 패턴

### Discriminated Union (판별 유니온)

공통 태그 필드로 케이스를 구분. 컴파일러가 `switch`/`if`에서 자동으로 좁혀줌.

```typescript
type ApiResult<T> =
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function render(r: ApiResult<User>) {
  switch (r.status) {
    case 'loading': return 'Loading...';
    case 'success': return r.data.name;  // ✅ data 접근 가능
    case 'error':   return r.error;
  }
}
```

비교 — 옵셔널만으로 표현하면(`data?: T; error?: string`) 매번 null 체크 필요 + 잘못된 조합도 가능. 판별 유니온은 **불가능한 상태를 타입으로 막는** 게 본질.

### Type Guards

| 종류 | 예시 |
|------|------|
| `typeof` | `typeof x === 'string'` |
| `instanceof` | `err instanceof Error` |
| `in` | `'meow' in animal` |
| User-defined predicate | `function isUser(v: unknown): v is User { ... }` |
| `asserts` 함수 | `function assert(v): asserts v is string { ... }` |

User-defined는 함수 본문이 진짜로 검증하는지 컴파일러가 확인 안 함 → 실수하면 거짓말이 됨. 그래서 보통 zod 같은 검증 라이브러리에 위임.

### `as const` (Const Assertion)

리터럴 값을 가장 좁은 리터럴 타입으로 고정 + 모든 속성 readonly.

```typescript
const ROLES = ['admin', 'user', 'guest'] as const;
type Role = typeof ROLES[number]; // 'admin' | 'user' | 'guest'

const STATUS = { IDLE: 'idle', LOADING: 'loading' } as const;
type Status = typeof STATUS[keyof typeof STATUS]; // 'idle' | 'loading'
```

**`enum` 대신 `as const` 객체**가 권장되는 이유:

| 항목 | `enum` | `as const` |
|------|--------|------------|
| 트리 쉐이킹 | 어려움 | OK |
| 런타임 코드 생성 | 있음 | 없음 |
| JS 호환성 | 별개 구조체 | 그냥 객체 |

### 모듈 — `import type` / `declare module`

```typescript
import type { User } from './types';      // 빌드 시 완전히 사라짐 → 트리 쉐이킹 보장
import { fn, type ApiResponse } from './api';
```

`verbatimModuleSyntax: true` 옵션과 함께 쓰면 ESM/CJS 호환 문제를 줄여줌.

## 컴파일러 옵션

### 항상 켜야 하는 것

```json
{ "strict": true }
```

`strict`가 켜는 묶음:

| 옵션 | 효과 |
|------|------|
| `noImplicitAny` | 추론 실패 시 any 자동 주입 금지 |
| `strictNullChecks` | null/undefined 다른 타입에 자동 할당 불가 |
| `strictFunctionTypes` | 파라미터 타입 체크 엄격화 (contravariance) |
| `strictBindCallApply` | `.bind/.call/.apply` 인자 타입 체크 |
| `strictPropertyInitialization` | 클래스 필드 생성자 초기화 강제 |
| `noImplicitThis` | 암묵적 this any 금지 |
| `alwaysStrict` | `"use strict"` emit |

### `strict` 묶음 외 권장

| 옵션 | 효과 |
|------|------|
| `noUncheckedIndexedAccess` | `arr[i]`가 `T \| undefined`로 추론 → 런타임 안전성 ↑ |
| `exactOptionalPropertyTypes` | `name?: string`이 정확히 "키가 있거나 없거나"를 의미 (값으로 undefined 명시는 별도) |
| `verbatimModuleSyntax` | `import type` 정확히 쓰도록 강제 |
| `isolatedModules` | 파일 단위 트랜스파일 가능하도록 (Vite/esbuild/swc 호환) |

### 권장 baseline (2026 기준)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

`@tsconfig/strictest` npm 패키지가 비슷한 baseline을 제공.

## TypeScript 6 / 7

### 6.x — Bridge Release

5.x → 7.0(Native)으로 가는 전환기. 큰 신기능 없이 **마이그레이션 준비**에 초점.

- 7.0에서 빠질 컴파일러 API를 deprecated 표시
- 사실상 마지막 JS 구현 메이저
- 5.x와 미묘하게 달랐던 엣지 케이스 시맨틱 통일
- **앱 코드는 거의 영향 없음**

### 7.0 — TypeScript Native (Go 기반)

2025년 3월 Microsoft 발표. **컴파일러를 Go로 포팅**.

| 영역 | 5.x/6.x | 7.0 |
|------|---------|-----|
| 구현 언어 | TypeScript | **Go** |
| 실행 환경 | Node.js | **단일 네이티브 바이너리** |
| 빌드 속도 | 기준 | **~10배** |
| 메모리 | 기준 | **~50%** |
| 병렬 처리 | 제한적 | **goroutine 기반** |
| LSP | TS 서버 (Node) | **Go 기반 LSP** |

**왜 Go인가 (Rust 아니라)** — Anders Hejlsberg 답변: 기존 JS 코드의 메모리 모델(GC, 객체 그래프, 사이클)이 Go와 직관적으로 매핑됨. Rust의 ownership/borrow checker로는 1:1 포팅이 거의 불가능 → 처음부터 다시 써야 함. Go 선택으로 약 1년 만에 포팅 가능.

**시맨틱 호환성** — 100% 동일하도록 설계. 같은 코드 = 같은 에러/통과. Drop-in replacement 목표.

**깨질 가능성**:
- `ts-morph` (컴파일러 API 래퍼)
- `@typescript-eslint`의 type-aware 룰
- 일부 커스텀 transformer (NestJS 데코레이터, `tsc-alias`, `ts-patch` 류)
- `tsx`/`esbuild`/`swc`/`Vite` 등 자체 파서 사용 도구는 **거의 영향 없음**

**개발자가 할 일**: 앱 코드는 그대로. `npm install -D typescript@7` 하면 빌드만 빨라짐. 라이브러리/툴 메인테이너만 호환성 작업 필요.

> 위 5.x 기준 학습 내용은 **6/7에서도 그대로 유효**. 언어 시맨틱은 안 바뀜. 바뀌는 건 컴파일러 구현체.

## 핵심 질의응답

**Q. TS는 런타임에 타입을 검증해주는가?**
A. 아니다. `tsc` 결과물에서 타입은 모두 사라진다(type erasure). API 응답이 정의된 타입과 다르면 컴파일은 통과하고 런타임에 터진다. 시스템 경계(외부 입력)에서는 zod 같은 런타임 스키마 검증이 필요하다.

**Q. AI가 코드를 짜는 시대에도 타입이 의미 있는가?**
A. 오히려 더 중요해진다. (1) AI hallucination(존재하지 않는 메서드)을 빌드 타임에 잡아주는 정적 검증기 역할, (2) 타입 시그니처가 AI에게 주는 명세서 — 컨텍스트가 좁혀져 더 정확한 코드 생성, (3) 사람의 PR 리뷰 시 타입이 가장 빠른 이해 수단.

**Q. 구조적 타이핑의 단점은 어떻게 우회하나?**
A. Branded Types로 nominal 타이핑을 흉내낸다. `type UserId = string & { __brand: 'UserId' }` — `__brand`는 컴파일 타임에만 존재하지만 타입 시스템상 호환을 막는다. zod의 `.brand()`가 같은 효과.

**Q. `interface`와 `type` 중 무엇을 우선 써야 하는가?**
A. `interface`로 가능하면 `interface`. 안 되는 경우(Union, Intersection, Tuple, Mapped, Conditional)에만 `type`. 이유: `interface`만 선언 병합이 가능해서 외부 라이브러리 타입 확장에 필수, 그리고 named cache 덕에 대형 프로젝트에서 약간 더 빠르다.

**Q. `Pick`이 `Omit`보다 안전한 이유는?**
A. `Pick<T, K>`는 `K extends keyof T`라 오타가 즉시 컴파일 에러. `Omit<T, K>`는 `K extends keyof any`라 키가 T에 없어도 통과 → 잘못된 결과를 사일런트하게 반환한다.

**Q. `never`의 진짜 활용처는?**
A. Discriminated Union의 Exhaustiveness Check. `default` 자리에 `const _: never = value`를 두면 새 케이스 추가 시 그 자리에서 자동으로 컴파일 에러가 나서 처리 누락을 빌드 타임에 발견할 수 있다.

**Q. `enum`이 권장되지 않는 이유는?**
A. (1) 트리 쉐이킹이 어려움, (2) 런타임에 객체 코드를 생성해서 번들 사이즈가 늘어남, (3) JS 호환성 측면에서 별개 구조체. 대안은 `as const` 객체 + `typeof X[keyof typeof X]` 패턴.

**Q. TS 7.0이 Rust가 아닌 Go로 포팅된 이유는?**
A. 기존 JS 코드의 메모리 모델(GC, 객체 그래프, 순환 참조)이 Go와 직관적으로 매핑된다. Rust의 ownership/borrow checker로는 그래프 구조 처리가 어려워 1:1 포팅이 사실상 처음부터 다시 쓰는 셈이 된다. Go 선택으로 마이그레이션 비용을 약 10배 줄였다.

## 주의사항 / 자주 하는 실수

- **`as User` 같은 타입 단언을 검증으로 착각** — TS는 단언을 그냥 믿는다. 외부 입력은 zod로 검증.
- **`any` 남용** — 전염성으로 주변 코드 타입 안전성까지 무력화. ESLint `no-explicit-any` 룰로 차단.
- **`Omit`의 사일런트 통과** — 오타 시 빈 동작. 키가 적으면 `Pick`을 우선.
- **`interface`로 union 시도** — 안 됨. union/intersection은 `type`만 가능.
- **선언 병합으로 인한 의도치 않은 타입 변경** — 같은 이름의 `interface`를 누가 추가하면 조용히 합쳐짐. unique한 게 좋으면 `type` 사용.
- **`Readonly<T>`가 깊지 않음** — 얕은 readonly. 깊은 불변은 `DeepReadonly` 직접 정의 필요.
- **`exactOptionalPropertyTypes` 미적용** — `name?: string`에 `undefined`를 명시적으로 넣을 수 있는지가 옵션에 따라 달라짐. 의도를 정확히 표현하려면 켜는 편이 좋음.
- **타입 곡예(Type Gymnastics)** — 시그니처가 본문보다 길면 의심. 라이브러리에선 가치가 있지만 앱 코드에선 비용 > 안전성이 되기 쉬움.
- **`enum` 사용** — `as const` 객체로 대체.
- **`moduleResolution: "node"` 잔존** — 번들러 환경이면 `"bundler"`, Node ESM이면 `"nodenext"`로 갱신.

## 참고

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [TypeScript Roadmap (GitHub Wiki)](https://github.com/microsoft/TypeScript/wiki/Roadmap)
- [typescript-go (Native 포팅)](https://github.com/microsoft/typescript-go)
- [type-fest](https://github.com/sindresorhus/type-fest) — 더 엄격한 유틸리티 타입 모음
- [`@tsconfig/strictest`](https://www.npmjs.com/package/@tsconfig/strictest) — 권장 strict baseline
- [zod](https://zod.dev) — 런타임 스키마 검증 + 타입 추론
