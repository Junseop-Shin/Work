# GraphQL 기본

> 2026-05-07 | GraphQL, ApolloClient, ApolloServer, urql, DataLoader, Subscription, Schema, Resolver, REST비교

## 한 줄 요약

GraphQL은 **클라이언트가 응답 모양을 결정하는 API 패러다임**이다. REST의 over/under-fetching 문제를 해결하고 타입 시스템을 1급으로 끌어올린 게 핵심 가치. 다만 단일 클라이언트 풀스택 TS에서는 tRPC가 더 매끄럽고, 캐싱(서버측)·파일 업로드·subscription scaling이 모두 약점이라 **만능 키가 아님**. **다양한 클라이언트가 같은 백엔드를 공유할 때** 최강.

## 핵심 개념

### REST의 두 고질병

```
GET /users/123
→ { id, name, email, avatar, bio, followers, following, createdAt, ... }
```

**Over-fetching** — 모바일에 이름만 필요한데 50개 필드가 옴. 네트워크 낭비.

```
GET /users/123        → User
GET /users/123/posts  → 그 사람 게시물
GET /posts/456/likes  → 그 게시물 좋아요
```

**Under-fetching (N+1 round-trip)** — 한 화면에 필요한 데이터가 여러 엔드포인트에 분산 → 클라가 여러 번 round-trip. 모바일에서 더 치명적.

REST는 **서버가 응답 모양을 결정**한다. 클라이언트는 받아서 쓸 뿐.

---

### GraphQL의 발상 — 클라이언트가 모양을 결정

```graphql
query {
  user(id: "123") {
    name              # 이름만
    posts(limit: 5) { # 같은 요청에 게시물도
      title
      likes(top: 3) { # 게시물별 좋아요 상위 3개도
        user { name }
      }
    }
  }
}
```

응답이 **쿼리 모양과 동일한 형태**:

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "posts": [
        { "title": "Hello", "likes": [{ "user": { "name": "Bob" } }] }
      ]
    }
  }
}
```

**핵심 사상:**
- **단일 엔드포인트** — 보통 `POST /graphql` 하나. URL 디자인 안 함
- **클라이언트가 쿼리 모양 정의** — 필요한 필드만, 필요한 깊이만
- **응답 모양 = 쿼리 모양** — 예측 가능
- **타입 시스템이 계약** — 스키마(`schema.graphql`)가 진실의 원천

---

### 3대 구성요소 — Schema, Query, Resolver

**Schema (계약)**

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!         # 비어있을 수 있지만 null 아님
}

type Post {
  id: ID!
  title: String!
  author: User!           # 역참조 — User → Post → User 가능
  likes: [Like!]!
}

type Query {
  user(id: ID!): User
  posts(limit: Int = 10): [Post!]!
}

type Mutation {
  createPost(title: String!): Post!
}

type Subscription {
  newPost: Post!
}
```

**`!` 와 `[]` 의 의미:**
- `String` — null 가능
- `String!` — null 불가
- `[Post]` — 배열 자체 또는 원소가 null 가능
- `[Post!]` — 배열은 null 가능, 원소는 null 불가
- `[Post!]!` — 배열도 원소도 null 불가 (가장 엄격)

**Query (요청)**

```graphql
# 읽기
query GetUser($id: ID!) {
  user(id: $id) { name posts { title } }
}

# 쓰기
mutation CreatePost($title: String!) {
  createPost(title: $title) { id title }
}

# 실시간 (WebSocket/SSE)
subscription OnNewPost {
  newPost { id title }
}
```

3종류: `query` (읽기), `mutation` (쓰기), `subscription` (실시간 스트림).

**Resolver (실행 함수)**

```ts
const resolvers = {
  Query: {
    user: async (_, args, ctx) => {
      return await db.user.findUnique({ where: { id: args.id } })
    },
  },
  User: {
    // User.posts 필드는 이 리졸버가 채움
    posts: async (user, _, ctx) => {
      return await db.post.findMany({ where: { authorId: user.id } })
    },
  },
  Post: {
    author: async (post, _, ctx) => {
      return await db.user.findUnique({ where: { id: post.authorId } })
    },
  },
}
```

**리졸버는 트리 순회처럼 호출됨.** 쿼리에 적힌 필드마다 해당 리졸버. 강점은 유연성, 약점은 N+1 (아래).

---

### N+1 문제와 DataLoader

```graphql
query {
  posts(limit: 100) {
    title
    author { name }   # ← 100개 게시물의 작성자
  }
}
```

순진한 리졸버 호출:
1. `posts` → DB 1번 (`SELECT * FROM posts LIMIT 100`)
2. 각 post마다 `Post.author` → DB **100번** (`SELECT * FROM users WHERE id = ?`)

**총 101번 쿼리 — N+1 문제.**

**해결: DataLoader** (Facebook 패키지). 같은 tick의 ID 요청을 모아서 한 번에 + 요청 내 캐싱.

```ts
const userLoader = new DataLoader(async (userIds) => {
  const users = await db.user.findMany({ where: { id: { in: userIds } } })
  return userIds.map(id => users.find(u => u.id === id))
})

Post: {
  author: (post) => userLoader.load(post.authorId),  // 자동 배칭
}
```

→ DB 2번으로 끝 (posts 1번 + users 1번).

**GraphQL을 쓰면 DataLoader는 거의 필수.** 모르고 운영 가면 DB 폭발.

---

### 캐싱 3층 — REST보다 까다로움

```
┌─────────────────────────────────────────┐
│ 1. 클라이언트 정규화 캐시               │  Apollo Client / urql
│    (브라우저 메모리)                    │  같은 User#123 한 번만
└─────────────────────────────────────────┘
              ↕ HTTP
┌─────────────────────────────────────────┐
│ 2. 서버 응답 캐시 (선택, 까다로움)      │  Redis / CDN
│    @cacheControl, persisted queries     │
└─────────────────────────────────────────┘
              ↕ 리졸버
┌─────────────────────────────────────────┐
│ 3. DataLoader (요청 내 배칭/중복제거)   │  요청 1개 안에서만!
│    cross-request 아님                   │
└─────────────────────────────────────────┘
              ↕ DB
┌─────────────────────────────────────────┐
│ 4. DB 자체 메모리 캐시 (별개 층)        │  PostgreSQL shared_buffers 등
└─────────────────────────────────────────┘
```

**(1) 클라이언트 정규화 캐시 — Apollo의 핵심**

```
브라우저 메모리 (Apollo cache):
  User#123 → { id, name, email, ... }
  Post#456 → { id, title, authorId: "User#123" }
```

같은 User#123을 다른 쿼리가 참조해도 메모리에 1번만. `User#123.name` 변경 → 그 객체를 보는 모든 쿼리 결과 자동 동기화.

**TanStack Query와 비교** — TanStack은 쿼리 결과를 통째로 (key-value), 정규화 안 함. GraphQL이 정규화로 가는 이유는 응답이 깊이 중첩되고 같은 객체가 여러 위치에 등장하기 때문.

**(2) 서버 응답 캐시 — GraphQL의 약점**

REST는 URL이 키 → CDN/HTTP 캐시 자연스러움 (`Cache-Control: max-age=300`). GraphQL은 모든 요청이 `POST /graphql`이라 **CDN/HTTP 캐시가 안 먹는다**.

해결책:

**Persisted Queries** — 미리 등록한 쿼리 ID로 GET:
```
GET /graphql?id=abc123&variables={...}
→ CDN 캐시 가능
```
Apollo, Relay 지원. 운영 비용 추가.

**`@cacheControl` + Redis**:
```graphql
type User @cacheControl(maxAge: 300) {
  id: ID!
  name: String!
}
```
Apollo Server가 응답을 Redis에 저장. **invalidation이 어려움** (같은 객체가 여러 쿼리에 분산).

→ **REST보다 서버 캐시가 까다롭다**는 게 GraphQL의 큰 약점.

**(3) DataLoader는 캐시 아님**

요청 1개 안에서만 동작. **다음 요청에는 사라짐**. 그래서 "캐시"라기보단 "배치 처리기 + 요청 내 중복 제거기".

cross-request 캐시 안 하는 이유:
- 사용자별 권한 다름
- DB 일관성 (오래된 캐시 → 오래된 답)
- 메모리 폭증

→ **cross-request 캐시는 Redis/CDN이 별도 책임**.

---

### Subscription의 한계 — WebSocket Scaling

스펙은 있지만 **실무에서 거의 안 쓰는 부분**.

**Transport:**

| Transport | 양방향 | 비고 |
|---|---|---|
| WebSocket (`graphql-ws`) | ✅ | 가장 흔함, stateful |
| SSE (`graphql-sse`) | ❌ (서버→클라만) | 구독은 단방향이라 충분, 가벼움 |

**WebSocket Scaling 문제:**

서버 메모리에 클라 1명당 연결 객체 + 구독 상태 유지:

```
Node.js (ws): 10k~50k connection / 인스턴스
Erlang/Elixir (Phoenix): 200만+ / 인스턴스 (BEAM 특성)
일반 Java/Python: 보통 1만 미만
```

**여러 인스턴스 띄우면 또 다른 문제:**

```
사용자 A → 서버 1에 연결, "newPost" 구독
사용자 B → 서버 2에 연결, post 작성 (mutation)
                          ↓
                  서버 2 newPost 이벤트 발생
                          ↓
            서버 1은 이걸 어떻게 알지? ← 인스턴스 간 메시지 공유 필요
```

해결: **Redis Pub/Sub / Kafka / NATS** + sticky session 로드밸런서. 인프라 한 묶음 추가.

**그래서 실무에서는?**

- **GitHub, Shopify** GraphQL API → subscription 없음. 외부 알림은 webhook.
- **사내 서비스 + 진짜 실시간 필요** → 보통 **별도 WebSocket 서버 분리** 또는 Pusher/Ably 같은 관리형 서비스
- **거의 실시간** → polling 5-30초가 압도적으로 단순

**Subscription 결정 매트릭스:**

| 시나리오 | 추천 |
|---|---|
| 채팅, 협업 (Notion 동시 편집), 게임 | subscription 또는 자체 WebSocket |
| 알림 (좋아요, 댓글) | polling 또는 push notification |
| 외부 API 이벤트 알림 | webhook |
| 대시보드 (메트릭, 주가) | SSE |
| 100만+ 동시접속 | 관리형 서비스 또는 Erlang/Elixir |

→ **GraphQL subscription = "있긴 한데 거의 안 쓰는 부분".**

---

### 파일 업로드 — REST 별도가 일반적

GraphQL은 JSON 기반이라 multipart/form-data 바이너리와 모델이 안 맞음.

**A. REST 엔드포인트 따로 (가장 흔함)**
```
POST /upload (REST, multipart) → S3 → URL 반환
mutation createPost(imageUrl: ...) (GraphQL)
```

**B. graphql-multipart-request-spec**
GraphQL이 multipart 다루도록 표준 확장. **Apollo가 v3에서 지원 뺌** (보안/복잡도). graphql-yoga 등에서만 가능. 비표준 영역.

**C. Pre-signed URL (요즘 대세)**
```graphql
mutation {
  presignUpload(filename: "avatar.png") {
    uploadUrl    # ← S3가 발급한 직접 업로드 URL
    publicUrl
  }
}
```
1. 클라가 GraphQL로 pre-signed URL 받음
2. 그 URL로 **S3에 직접 PUT** (서버 안 거침)
3. 완료 후 GraphQL로 메타데이터 저장

서버 부하 ↓, 큰 파일 OK. AWS S3 / Cloudflare R2 / GCS 다 지원. **GraphQL 진영 권장 패턴**.

---

### 클라이언트 — Apollo Client / urql

```tsx
import { useQuery, gql } from "@apollo/client"

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      name
      posts { title }
    }
  }
`

function UserPage({ id }) {
  const { data, loading, error } = useQuery(GET_USER, { variables: { id } })
  if (loading) return <Spinner />
  return <h1>{data.user.name}</h1>
}
```

**핵심 기능:**
- **정규화 캐시** — 같은 객체 메모리에 1번
- **Optimistic UI** — mutation 끝나기 전에 화면 먼저 업데이트
- **자동 refetch** — mutation 후 관련 쿼리 무효화 (`refetchQueries`)
- **로컬 상태 통합** — Apollo는 클라 로컬 상태도 GraphQL로 다룰 수 있게 함 (Reactive Variables)

**urql** — Apollo 대안. 더 가볍고 단순. 정규화 캐시는 옵션 (기본은 document cache).

---

### codegen — 타입 안전성

```bash
graphql-codegen --config codegen.yml
```

`schema.graphql` + 클라 쿼리 파일 → TypeScript 타입 + 훅 자동 생성:

```ts
// 자동 생성됨
export function useGetUserQuery(options: { variables: { id: string } }) {
  return useQuery<GetUserQuery, GetUserQueryVariables>(GET_USER, options)
}

const { data } = useGetUserQuery({ variables: { id: "123" } })
data?.user?.name  // 타입 추론 완벽
```

**스키마 변경 → codegen 다시 → 깨진 곳 컴파일러가 잡아줌**. GraphQL의 진짜 가치 중 하나.

---

### 언어별 구현체 — Node 전용 아님

GraphQL은 **스펙(spec)이지 라이브러리가 아님**. 페이스북이 스펙 공개, 모든 주요 언어에 구현체 존재:

| 언어 | 구현체 |
|---|---|
| **Node.js / TS** | Apollo Server, graphql-yoga, Mercurius (Fastify), NestJS GraphQL 모듈 |
| **Python** | Strawberry (TS-stylish), Graphene, Ariadne |
| **Go** | gqlgen, graphql-go |
| **Java** | DGS (Netflix), graphql-java, Spring for GraphQL |
| **.NET** | Hot Chocolate, GraphQL.NET |
| **Ruby** | graphql-ruby (GitHub이 자기네 API에 사용) |
| **Rust** | async-graphql, juniper |
| **PHP** | webonyx/graphql-php |
| **Elixir** | Absinthe |

**실제 사례:**
- GitHub GraphQL API → Ruby (graphql-ruby)
- Netflix → Java (DGS)
- Shopify → Ruby
- Facebook (원조) → Hack/PHP

TS 진영이 도구가 가장 풍부 (Apollo, urql, graphql-codegen). 그래서 풀스택 TS면 자연스럽게 TS GraphQL.

GraphQL 스펙은 **wire format(요청/응답 형식)** 만 정의. 어떤 언어로 구현하든 자유.

---

### REST vs GraphQL 비교

| | REST | GraphQL |
|---|---|---|
| 엔드포인트 | 리소스마다 (`/users`, `/posts`) | 단일 (`/graphql`) |
| 응답 모양 | 서버 결정 | 클라이언트 결정 |
| Over/Under-fetching | 흔함 | 거의 없음 |
| 타입 계약 | OpenAPI 별도 | 내장 (스키마) |
| HTTP 캐시 (CDN) | ✅ 자연스러움 | ❌ POST라 안 됨 |
| 클라 캐시 | 단순 (RTK Query/TanStack) | 정규화 (Apollo, urql) — 강력하지만 복잡 |
| 서버 응답 캐시 | URL 기반 단순 | 객체 기반, invalidation 까다로움 |
| 요청 내 중복 제거 | 보통 안 함 | DataLoader 거의 필수 |
| 파일 업로드 | 자연스러움 (multipart) | 어색함 (pre-signed URL 또는 REST 별도) |
| 실시간 | webhook / WebSocket 별도 | subscription 내장 (단, scaling 약점) |
| 학습 비용 | 낮음 | 높음 |
| 디버깅 | curl/Postman 친화 | 전용 도구 (GraphiQL, Apollo Studio) |
| 표준 도구 친화 | 강함 | 약함 |

---

### 의사결정 — 언제 쓰나

✅ **쓰면 좋은 경우:**
- 다양한 클라이언트 (웹/iOS/Android/외부 통합) 가 같은 백엔드 공유 — 각자 모양이 다름
- 데이터 모양이 화면마다 크게 달라 over-fetching 비용이 큼
- 페이스북/GitHub/Shopify 같은 플랫폼적 API
- 백엔드가 여러 마이크로서비스를 통합하는 경계 (API Gateway 역할)

❌ **비추인 경우:**
- 단일 클라이언트 풀스택 TS — **tRPC**가 codegen도 없이 끝남
- 작은 팀 / MVP — 학습 비용 + DataLoader + 캐시 정책이 가치를 넘어섬
- 파일 업로드 위주 / 단순 CRUD — REST가 더 자연스러움
- 캐싱이 중요한 공개 API — HTTP 캐시 못 쓰는 게 큰 손해
- subscription에 의존하는 100만+ 동시접속 — 인프라 비용 폭발

→ **GraphQL = 클라이언트가 다양하고 데이터 형태가 복잡할 때.** 만능 키 아님.

---

### 검증 측면 — zod와의 관계

GraphQL 스키마는 **타입 검증** 자동:

```graphql
type Query {
  user(id: ID!): User      # id 없거나 ID 형식 아니면 거절
}
```

하지만 **비즈니스 룰** (이메일 형식, 비밀번호 길이, 날짜 범위)은 별도:

```ts
Mutation: {
  signup: async (_, { input }) => {
    const result = SignupSchema.safeParse(input)  // zod로 한 번 더
    if (!result.success) throw new GraphQLError(...)
    return await db.user.create({ data: result.data })
  },
}
```

**풀스택 TS + GraphQL 조합에서도 zod 살아있음** — 스키마 타입은 GraphQL이, 비즈니스 룰은 zod가.

---

## 핵심 질의응답

**Q. 단일 엔드포인트 (`POST /graphql`)는 무슨 의미인가?**
A. 모든 요청이 같은 URL. URL로 리소스 구분 안 하고 body의 쿼리로 구분. 그래서 HTTP 캐시(CDN) 못 먹음.

**Q. GraphQL은 Node.js 전용인가?**
A. 아니다. GraphQL은 스펙. 모든 주요 언어 구현체 존재 (GitHub은 Ruby, Netflix는 Java). TS 도구 생태계가 풍부할 뿐.

**Q. N+1 문제는 GraphQL만의 문제인가?**
A. 아니다. ORM 일반의 문제. 다만 GraphQL은 클라이언트가 깊이를 결정해서 발생 가능성이 더 흔함. DataLoader가 거의 필수.

**Q. DataLoader는 cross-request 캐시인가?**
A. 아니다. 요청 1개 안에서만 배칭/중복제거. 다음 요청에는 사라짐. cross-request 캐시는 Redis 등 별도 책임.

**Q. Apollo 정규화 캐시 vs TanStack Query 차이?**
A. TanStack은 key-value (쿼리 결과 통째로). Apollo는 객체 단위 정규화 (User#123 한 번만). 응답이 깊이 중첩되는 GraphQL 특성에 정규화가 자연스러움.

**Q. Subscription을 WebSocket으로 하면 유저 많아질 때 안 되지 않나?**
A. 정확. 이게 GraphQL의 큰 약점. stateful 연결이라 한 인스턴스에 한계 (Node.js 1만~5만), 수평 확장 시 Redis Pub/Sub 같은 인프라 필요. 그래서 GitHub/Shopify 같은 곳도 subscription 안 쓰고 webhook 사용.

**Q. 파일 업로드는 어떻게?**
A. GraphQL은 JSON이라 어색함. 보통 (a) REST 엔드포인트 별도, (b) Pre-signed URL로 S3 직접 업로드 후 GraphQL로 메타데이터. (b)가 권장.

**Q. 풀스택 TS 단일 클라이언트인데 GraphQL 써야 하나?**
A. 비추. tRPC가 codegen도 없이 더 매끄러움. GraphQL은 클라가 다양할 때 빛남.

**Q. GraphQL 스키마의 타입 검증으로 zod 같은 거 필요 없지 않나?**
A. 타입(string/int/non-null)은 GraphQL이 검증. 비즈니스 룰(이메일 형식, 길이)은 별도. 보통 리졸버 안에서 zod로 한 번 더.

**Q. `!`와 `[]`의 정확한 의미?**
A. `!`는 non-null. 배열은 두 군데 있음 — `[Post!]!` 는 배열도 원소도 null 불가 (가장 엄격). 안 붙이면 null 가능.

**Q. CDN 캐시를 쓸 방법은 정말 없나?**
A. Persisted Queries로 GET + URL 기반으로 우회 가능. 운영 비용 추가. Apollo, Relay 지원.

## 주의사항 / 자주 하는 실수

- **DataLoader 없이 운영** — 트래픽 늘면 DB 폭발. N+1 쿼리 폭증. GraphQL 도입 시 DataLoader는 옵션 아니라 기본.
- **subscription을 단순 알림에 쓰기** — WebSocket 인프라 부담이 polling/webhook보다 압도적. 진짜 실시간 (채팅, 동시 편집) 아니면 polling.
- **Apollo 캐시 정책 무시** — `cache.evict`, `refetchQueries`, `update` 안 쓰면 mutation 후 화면 안 바뀜. 정규화 캐시는 강력하지만 정확히 알아야 동작.
- **GraphQL을 RPC처럼 쓰기** — Mutation에 `doSomething(input: X): Result` 만 잔뜩 만들면 GraphQL 가치(필드 선택) 무용. 그럴 거면 tRPC.
- **공개 API에서 `@cacheControl` 안 걸기** — CDN 캐시 못 쓰는 GraphQL은 캐싱이 엄격한 부담. 큰 트래픽이면 persisted queries + CDN 필수.
- **외부 API에 webhook 대신 subscription 노출** — 외부 클라가 WebSocket 유지해야 하는데 비현실적. 외부 알림은 webhook으로.
- **파일 업로드 GraphQL로 multipart 시도** — Apollo는 v3에서 지원 뺌. 비표준에 의존하지 말고 pre-signed URL 패턴.
- **persisted queries 없이 GraphQL을 CDN 뒤에 두기** — 모든 요청이 origin까지 가서 origin 부담 폭증.
- **DataLoader 캐시를 cross-request로 쓰려는 실수** — 권한 다른 사용자가 공유 캐시 보면 보안 사고. 항상 요청 단위로.

## 참고

- [GraphQL 공식 스펙](https://graphql.org/)
- [Apollo Server](https://www.apollographql.com/docs/apollo-server/)
- [Apollo Client](https://www.apollographql.com/docs/react/)
- [urql](https://commerce.nearform.com/open-source/urql/)
- [graphql-yoga](https://the-guild.dev/graphql/yoga-server)
- [graphql-codegen](https://the-guild.dev/graphql/codegen)
- [DataLoader](https://github.com/graphql/dataloader)
- [graphql-ws (subscription transport)](https://github.com/enisdenjo/graphql-ws)
- [graphql-sse](https://github.com/enisdenjo/graphql-sse)
- 관련 학습: [프론트-백엔드 계약 동기화](../아키텍처/프론트-백엔드-계약-동기화.md) — GraphQL이 동기화 전략 중 하나로 등장
- 관련 학습: [폼 라이브러리 + 검증 비교](../폼-유효성검사/폼-라이브러리-검증-비교.md) — zod의 비즈니스 룰 검증 측면
- 관련 학습: [서버 상태 라이브러리 비교](../상태관리/서버상태-라이브러리비교.md) — TanStack Query vs Apollo 캐시 모델 차이
