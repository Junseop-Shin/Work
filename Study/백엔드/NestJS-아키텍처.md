# NestJS 아키텍처 — DI / 모듈 / 요청 생명주기

> 2026-06-02 | NestJS, DI, IoC, 데코레이터, 모듈, Guard, Interceptor, Pipe, 이벤트루프, Spring비교

## 한 줄 요약

NestJS는 **Spring/Angular의 DI·계층 구조를 TypeScript에 강제한 "메타프레임워크"**다. 자기가 HTTP를 처리하지 않고 Express/Fastify를 어댑터로 감싸며, Express의 "무구조" 문제를 IoC 컨테이너·모듈 캡슐화·정해진 요청 생명주기로 해결한다. 사상은 Spring과 거의 1:1이지만, TS는 타입이 소거되므로 `emitDecoratorMetadata`로 타입을 메타데이터에 박아 DI를 구현한다.

> 프레임워크 비교(Express/Koa/Fastify/Spring/FastAPI/Django)는 별도 문서 → [백엔드 프레임워크 비교](백엔드-프레임워크-비교.md)

## 핵심 개념

### 왜 NestJS를 쓰는가 — Express에는 "구조"가 없다

Express는 프레임워크라기보다 **라우팅 + 미들웨어 최소 키트**다. "이 파일은 컨트롤러, 저긴 서비스, 의존성은 이렇게 주입해라"를 하나도 강제하지 않는다.

```js
// Express — 동작은 하지만 구조는 전적으로 개발자 몫
app.get('/users/:id', (req, res) => {
  const user = db.findUser(req.params.id);  // DB 접근을 핸들러에서 직접
  res.json(user);
});
```

→ 팀마다 구조 제각각, 로직/라우팅/데이터접근 뒤섞임, `req`/`res` 통째 모킹 없이는 테스트 곤란. 프로젝트가 커지면 카오스. (순수 Servlet으로 큰 서버 짜는 느낌)

NestJS의 답: **Angular(표면 문법)** + **Spring(서버 아키텍처 철학)**. 창시자가 Angular 출신이라 데코레이터 문법은 Angular를 닮았지만, DI·계층 구조는 Spring과 같다.

| Spring | NestJS | 역할 |
|--------|--------|------|
| `@RestController` | `@Controller` | HTTP 진입점 |
| `@Service` / `@Component` | `@Injectable` (Provider) | 비즈니스 로직, DI 대상 |
| `@Autowired` (생성자 주입) | 생성자 파라미터 주입 | 의존성 주입 |
| `@Configuration` 모듈 단위 | `@Module` | 컴포넌트 묶음 |
| `HandlerInterceptor` | `Interceptor` | 요청 전후 가로채기 |
| `Filter` / Spring Security | `Guard` | 인증/인가 |
| `@ControllerAdvice` | `Exception Filter` | 예외 처리 |
| `@Valid` + Bean Validation | `Pipe` + class-validator | 검증/변환 |
| `ApplicationContext` | IoC Container | 빈/프로바이더 컨테이너 |

### NestJS는 "프레임워크 위의 프레임워크"

NestJS는 HTTP 서버를 직접 구현하지 않는다. 기본은 Express, 옵션으로 Fastify를 **어댑터로 갈아끼운다.**

```
┌─────────────────────────────────────┐
│   NestJS (구조 / DI / 데코레이터)      │  ← 작성하는 추상 계층
├─────────────────────────────────────┤
│   Platform Adapter                   │  ← @nestjs/platform-express
│                                      │     or @nestjs/platform-fastify
├─────────────────────────────────────┤
│   Express  /  Fastify                │  ← 실제 HTTP 처리 엔진
└─────────────────────────────────────┘
```

→ NestJS는 Express의 경쟁자가 아니라 **상위 계층**. Spring MVC가 Tomcat/Undertow를 갈아끼우는 것과 같은 구조.

---

## DI 컨테이너 / IoC — `new` 없이 객체가 주입되는 원리

### IoC와 DI

```typescript
@Injectable()
export class UserService {
  constructor(private readonly repo: UserRepository) {}  // Spring 생성자 주입과 판박이
}
```

- **IoC(제어의 역전)** = 객체 생성/연결의 책임을 컨테이너에 넘김 (큰 원칙)
- **DI(의존성 주입)** = IoC를 구현하는 구체적 방법 (생성자/필드 주입)

`@Injectable()` = Spring `@Service`. `@Autowired`조차 불필요 — TS가 생성자 파라미터 타입을 읽으니까.

### 핵심: TS는 타입이 소거되는데 어떻게 타입을 아는가?

```typescript
constructor(private readonly repo: UserRepository) {}
```

문제: **TypeScript 타입은 컴파일되면 사라진다(type erasure).** 그런데 런타임에 컨테이너가 "이 자리에 `UserRepository`를 넣어야지"를 알아야 한다.

**답: `reflect-metadata` + `emitDecoratorMetadata`**

```json
// tsconfig.json
{
  "compilerOptions": {
    "experimentalDecorators": true,   // 데코레이터 문법 허용
    "emitDecoratorMetadata": true     // ★ 타입 정보를 메타데이터로 방출
  }
}
```

`emitDecoratorMetadata: true`면, **데코레이터가 붙은 클래스에 한해** 타입을 지우지 않고 메타데이터로 남긴다. 컴파일 결과(개념적):

```js
UserService = __decorate([
  Injectable(),
  __metadata("design:paramtypes", [UserRepository])  // ★ 생성자 파라미터 타입 = 실제 클래스 참조
], UserService);
```

부팅 시 NestJS 컨테이너:

```js
const paramTypes = Reflect.getMetadata('design:paramtypes', UserService); // [UserRepository]
const deps = paramTypes.map(t => container.resolve(t));
const instance = new UserService(...deps);  // 컨테이너가 대신 new
```

**Spring과의 차이:** Java는 JVM 리플렉션이 타입을 보존해 거저 되지만, TS는 타입이 소거되므로 "데코레이터 붙은 곳에만" 타입을 메타데이터로 남기는 트릭을 쓴다. → NestJS가 데코레이터에 의존하는 근본 이유.

> 의존성이 없는 클래스에도 `@Injectable()`을 붙여야 하는 이유: 데코레이터가 0개면 그 클래스의 `design:paramtypes`가 아예 방출되지 않아, 컨테이너가 "주입 가능한 놈인지"조차 모른다.

### 함정 ① — 인터페이스는 주입 불가

```typescript
constructor(private repo: IUserRepository) {}  // ❌ 인터페이스는 런타임에 사라짐 → Object로만 박힘
```

해결 — 명시적 토큰:

```typescript
constructor(@Inject('USER_REPO') private repo: IUserRepository) {}
// 모듈: providers: [{ provide: 'USER_REPO', useClass: PostgresUserRepository }]
```

클래스는 "클래스 자체"가 토큰이지만, 인터페이스/추상은 문자열·심볼 토큰을 직접 줘야 한다. (Spring은 인터페이스 타입 + `@Qualifier`로 푸는 지점)

### Provider 4가지 방식

```typescript
providers: [
  UserService,                                        // ① 표준 (클래스 = 토큰)
  { provide: 'CONFIG', useValue: { port: 3000 } },    // ② 상수/객체 (≈ @Value)
  { provide: UserRepo, useClass: PgUserRepo },        // ③ 토큰↔구현 교체 (테스트 시 mock)
  { provide: 'ASYNC', useFactory: async () => {...} },// ④ 동적/비동기 생성 (≈ @Bean 메서드)
]
```

`useClass` 교체는 Spring `@MockBean`과 같은 발상 — 테스트 시 `{ provide: UserRepo, useClass: MockUserRepo }`.

### 함정 ② — Injection Scope (기본은 싱글톤)

| NestJS Scope | Spring 대응 | 의미 |
|--------------|-------------|------|
| `DEFAULT` | `singleton` | 앱 전체 1개 (기본) |
| `REQUEST` | `request` | HTTP 요청당 1개 |
| `TRANSIENT` | `prototype` | 주입 지점마다 새로 |

`REQUEST` 스코프는 매 요청 의존성 트리를 새로 만들어 **성능 비용**이 크고, 이를 주입받은 상위 provider도 연쇄적으로 REQUEST로 **오염(bubble up)**된다. 꼭 필요할 때만.

---

## 모듈 시스템 — 의존성의 "경계선"

Spring은 하나의 거대한 `ApplicationContext`에 빈이 평평하게 들어가 전역에서 다 보인다. **NestJS는 모듈마다 독립된 DI 스코프(경계)를 갖는다.**

```typescript
@Module({
  imports: [DatabaseModule],      // 다른 모듈을 끌어옴
  controllers: [UserController],  // 라우트
  providers: [UserService],       // 모듈 내부 전용
  exports: [UserService],         // ★ 외부에 공개할 provider
})
export class UserModule {}
```

### 캡슐화 규칙 (= Java public / package-private)

```
┌─ UserModule ──────────────┐
│  providers: [UserService,  │
│              HashService]  │  ← 둘 다 모듈 내부에서 사용 가능
│  exports:   [UserService]  │  ← UserService만 외부 공개
└────────────────────────────┘
         │ exports
         ▼
┌─ OrderModule (imports: [UserModule]) ─┐
│  UserService 주입 ✅ / HashService ❌  │  (export 안 했으니)
└────────────────────────────────────────┘
```

1. A가 B의 provider를 쓰려면 → B가 `exports` + A가 `imports`
2. `exports` 안 한 provider는 모듈 내부 전용 (private)
3. **import는 전이되지 않는다** — A→B→C일 때 A는 C의 export를 자동으로 못 봄

→ 모듈 경계가 곧 아키텍처 문서이자 마이크로서비스 분리선. Spring의 "다 전역, 다 보임"과 정반대 철학(의존성을 코드에 명시).

### 모듈의 종류

```typescript
@Module({ ... })  export class UserModule {}        // ① 기능 모듈 (기본)

@Global()                                            // ② 전역 모듈 (import 없이 어디서나)
@Module({ providers: [ConfigService], exports: [ConfigService] })
export class ConfigModule {}

@Module({})                                          // ③ 동적 모듈 (설정 주입)
export class DatabaseModule {
  static forRoot(opts: DbOptions): DynamicModule {
    return { module: DatabaseModule,
             providers: [{ provide: 'DB_OPTIONS', useValue: opts }],
             exports: ['DB_OPTIONS'] };
  }
}
// imports: [DatabaseModule.forRoot({ host: '...' })]
```

- `forRoot`(앱당 1회, 전역 설정) / `forFeature`(모듈별 부분 설정) = Spring Boot `@EnableXxx` 스타터 패턴. (`TypeOrmModule.forRoot(...)` / `.forFeature([User])`)
- **`@Global()`과 일반 모듈은 동작·성능이 동일**(둘 다 싱글톤 1개). 차이는 "전역 가시성(편의) vs 명시적 import(의존성 추적)"뿐. 코드 스플리팅과 무관(서버는 프론트와 달리 코드를 다 메모리에 로드). 남용하면 명시적 경계 장점을 스스로 버림.

### 함정 — 순환 의존성

A가 B를, B가 A를 주입받으면 "먼저 만들 수 없는" 교착. `forwardRef`로 해석을 미룬다:

```typescript
constructor(@Inject(forwardRef(() => AuthService)) private auth: AuthService) {}
```

단 `forwardRef`가 보이면 **설계 냄새** — 보통은 공유 로직을 제3의 서비스로 추출해 순환을 끊는 게 정답.

---

## 요청 생명주기 (Request Lifecycle) — 실무의 핵심

```
요청
 │
 ▼ ① Middleware           (Express 미들웨어 — 가장 바깥)
 ▼ ② Guards               (인증/인가 — "들어올 자격 있나?")
 ▼ ③ Interceptors (before)(요청 가공 / 타이머 시작)
 ▼ ④ Pipes                (검증/변환 — "입력 올바른가?")
 ▼ ⑤ Route Handler        (★ 컨트롤러 메서드 = 비즈니스 로직)
 ▼ ⑥ Interceptors (after) (응답 가공 / 타이머 종료)
 ▼ ⑦ 응답
        └─ 예외 발생 시 → ⑧ Exception Filter
```

**왜 Guard(②)가 Pipe(④)보다 먼저인가:** 인증 안 된 요청에 검증·변환을 돌리는 건 낭비. "들어올 자격" 먼저, "입력 검증"은 그 다음.

### Spring MVC 요청 흐름과 매핑

| 순서 | NestJS | Spring MVC |
|------|--------|-----------|
| 1 | Middleware | Servlet Filter |
| 2 | **Guard** | Spring Security Filter + `@PreAuthorize` |
| 3 | Interceptor (before) | HandlerInterceptor `preHandle` |
| 4 | Pipe | Argument Resolver + `@Valid` |
| 5 | Handler | Controller 메서드 |
| 6 | Interceptor (after) | HandlerInterceptor `postHandle` |
| 7 | Exception Filter | `@ControllerAdvice` |

> Servlet Filter는 Servlet 컨테이너 안이지만 **DispatcherServlet보다 앞**에서 동작. Spring Security는 그 Filter로 구현돼 있어 "서블릿 디스패처 전단"에서 URL 기반 1차 인가를 하고, 메서드 레벨 인가는 `@PreAuthorize`(AOP, 핸들러 직전)로 별도 수행. **NestJS Guard는 이 둘을 한 군데(핸들러 직전, `ExecutionContext`로 핸들러를 이미 앎)에서 통합**한다.

### Guard — "통과/차단" (boolean)

```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler()); // 핸들러 메타데이터 읽기
    const { user } = context.switchToHttp().getRequest();
    return roles.some(r => user.roles.includes(r));  // false면 403, 핸들러 실행 안 됨
  }
}
// @Roles('admin') @Get('users') findAll() {}
```

- `ExecutionContext`로 "어떤 핸들러로 가는 요청인지" 메타데이터까지 읽어 인가 결정 → Spring Security 필터 + `@PreAuthorize` 통합본
- 인증/인가는 **Guard로** (Interceptor 아님): "차단(boolean)"이라는 단일 책임에 맞는 추상. `return false` 한 줄로 NestJS가 403까지 처리.
- 실무: `@nestjs/passport`의 `AuthGuard('jwt')`로 Passport 전략을 Guard로 실행. **`Spring Security ≈ Passport + AuthGuard`** (둘 다 검증된 전략을 표준 추상에 끼워 씀)

### Interceptor — RxJS로 핸들러를 감싸는 AOP (≈ Spring `@Around`)

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now();
    return next.handle().pipe(            // next.handle() = 핸들러 실행 (= joinPoint.proceed())
      map(data => ({ data, timestamp: now })),       // 응답 변형
      tap(() => console.log(`${Date.now() - now}ms`)), // 후처리
    );
  }
}
```

`next.handle()` 전 = before, 후 = after. Spring `HandlerInterceptor`(pre/post 분리)보다 **AOP `@Around`에 더 가깝다.** 용도: 응답 래핑, 실행시간 로깅, 캐싱, 타임아웃.

### Pipe — 검증 & 변환 (≈ Spring `@Valid` + Bean Validation)

```typescript
export class CreateUserDto {
  @IsEmail() email: string;
  @IsString() @MinLength(8) password: string;
  @IsInt() @Min(0) age: number;
}

@Post()
create(@Body() dto: CreateUserDto) {}  // ValidationPipe가 자동 검증 → 통과 = 검증된 데이터 보장
```

| | Spring | NestJS |
|---|--------|--------|
| 규칙 선언 | `@NotNull`, `@Email` (Hibernate Validator) | `@IsEmail`, `@MinLength` (class-validator) |
| 실패 시 | `MethodArgumentNotValidException` → 400 | `BadRequestException` → 400 |

**함정 — `class-transformer` + `whitelist` 필수:**

```typescript
app.useGlobalPipes(new ValidationPipe({
  transform: true,            // plain JSON → DTO 인스턴스 (이게 있어야 class-validator가 메타데이터를 읽음)
  whitelist: true,            // ★ DTO에 없는 속성 제거 → over-posting(권한 상승) 방어
  forbidNonWhitelisted: true, // DTO에 없는 속성 오면 에러
}));
```

`@Body()`로 온 건 그냥 plain object(클래스 정보 없음)라, `class-transformer`로 DTO 인스턴스로 바꿔야 데코레이터 메타데이터를 읽는다. `whitelist:false`(기본)면 `{ email, password, isAdmin:true }` 같은 over-posting이 그대로 통과 → **반드시 `whitelist:true`**.

### Exception Filter — 예외를 응답으로 (≈ `@ControllerAdvice`)

```typescript
throw new NotFoundException('User not found');  // → 404 (내장 글로벌 필터가 JSON 응답 자동 생성)

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();
    res.status(exception.getStatus()).json({ message: exception.message });
  }
}
```

함정: 핸들러에서 예외가 나면 **Interceptor의 after(`tap`/`map`)는 실행되지 않고** 곧장 Exception Filter로 간다(RxJS 스트림에 에러). "반드시 실행돼야 하는 후처리"는 `finalize()`를 써야 예외에도 실행됨.

> Guard/Interceptor/Pipe/Filter는 모두 **전역 / 컨트롤러 / 메서드 / 파라미터** 레벨로 적용 범위 선택 가능.

---

## 동시성 · 배포 모델 — 단일 스레드를 어떻게 굴리나

### 단일 스레드 ≠ 처리 못함

"단일 스레드"는 **JS 실행 스레드가 하나**라는 뜻이지 IO까지 한 줄로 기다린다는 게 아니다. IO(DB/네트워크/파일)는 **libuv + OS 커널(epoll/kqueue)이 백그라운드에서 논블로킹**으로 처리하고, JS 스레드는 결과를 기다리지 않고 다른 요청을 집어든다.

```
요청 A: DB await ──┐ (JS 스레드는 안 놂)
요청 B 처리 ───────┤
요청 C 처리 ───────┘→ A 응답 도착 시 콜백 큐 → A 재개
```

- 콜백은 **완료된 순서**로 큐에 들어가 **한 번에 하나씩, 끼어듦 없이(원자적)** 실행 → 락이 거의 불필요
- **IO-bound 워크로드(웹 백엔드 대부분 = DB 기다리기)에 강함.** 컨텍스트 스위칭 없이 수천 연결 처리
- **약점은 CPU-bound** — 동기 루프/암호화/이미지처리로 이벤트 루프를 블로킹하면 모든 요청 정지. 대응: `worker_threads`, 작업 큐(BullMQ), 별도 서비스 분리

### async/await = stackless coroutine (협력적 멀티태스킹)

`await`가 코루틴의 yield(양보점). "IO 기다릴 테니 제어권 놓을게, 결과 오면 깨워줘". 컨텍스트는 상태머신으로 보존됐다 재개.

| | JS async/await | Python asyncio | Kotlin coroutine | Go goroutine |
|---|---|---|---|---|
| 양보 | 협력적(`await`) | 협력적(`await`) | 협력적(`suspend`) | 협력적 |
| 스레드 | **단일 고정** | **단일(GIL)** | 디스패처로 멀티 가능 | **M:N 멀티코어** |
| 멀티코어 | ❌ | ❌ | ⭕ | ⭕ |

- **가장 정확한 1:1 대응 = Python asyncio** (단일 스레드 이벤트 루프 + stackless coroutine). FastAPI `async def`가 NestJS `async` 핸들러와 동작 모델 동일
- Go goroutine은 비슷해 보여도 런타임이 여러 OS 스레드에 분배(M:N)해 **진짜 멀티코어 병렬** → Node처럼 프로세스 다중화 불필요
- **협력적 모델의 숙명:** `await`(양보점)가 없으면 그 작업이 끝날 때까지 독점 → CPU-bound 약점의 근본 원인. (선점형 OS 스레드는 강제로 시간 분할)

### 멀티코어 활용 = 프로세스 N개

Node 한 프로세스는 1코어만 쓴다. → **스레드 대신 프로세스로 멀티코어 확보.**

| 층위 | 도구 | Spring 대비 |
|------|------|------------|
| 머신 내 멀티코어 | Node `cluster` / PM2 cluster mode | Spring은 1 JVM 멀티스레드 / Node는 코어 수만큼 프로세스 fork |
| 여러 머신 | K8s replicas / 오토스케일 | 동일 |
| 앞단 분배 | nginx / ALB | 동일 |

### DB 커넥션 풀 — HikariCP는 "사라진 게 아니라 ORM에 내장"

| Java | Node |
|------|------|
| HikariCP | `pg`의 `Pool`, `mysql2` pool |
| Spring Data JPA가 풀 관리 | TypeORM / Prisma 내부 풀 (`extra: { max: 10 }`) |

**Node 특유의 함정 — 커넥션 폭발:** 풀은 **프로세스마다 독립**. PM2로 8 프로세스 × `max:10` = 80 커넥션, K8s replica 10개까지 곱하면 800 → PostgreSQL `max_connections`(기본 100) 초과.

**해결 — 앞에 커넥션 풀러:**

```
Node 프로세스들 (각자 작은 풀) → PgBouncer (수백 클라 → 소수 DB 커넥션 멀티플렉싱) → PostgreSQL
```

`PgBouncer`는 특히 서버리스(프로세스 폭증)에서 거의 필수.

### 종합 — Spring 대응표

```
[클라이언트] → [nginx/ALB] → [Node 인스턴스 N개(각자 DB풀)] → [PgBouncer] → [PostgreSQL]
```

| Spring | Node/NestJS |
|--------|-------------|
| Tomcat 스레드풀 (동시 요청) | 이벤트 루프(IO) + 프로세스 클러스터(코어) |
| 스레드풀 크기 튜닝 | "이벤트 루프를 블로킹하지 마라" + 프로세스 수 |
| HikariCP | ORM/드라이버 내장 풀 (`pg.Pool`) |
| (JVM이라 불필요) | **PgBouncer** — 프로세스 다중화로 생긴 커넥션 폭발 방지 |

---

## 핵심 질의응답

**Q. `new` 없이 객체가 주입되는 원리는? Spring과 뭐가 다른가?**
A. 컨테이너가 생성자 파라미터 타입을 읽어 의존성을 채워 `new`해준다(IoC). Spring은 JVM 리플렉션으로 타입을 거저 알지만, TS는 타입이 소거되므로 `emitDecoratorMetadata`가 데코레이터 붙은 클래스의 타입을 `design:paramtypes` 메타데이터로 박아두고, NestJS가 `reflect-metadata`로 읽는다.

**Q. 왜 인터페이스로는 주입이 안 되나?**
A. 인터페이스는 런타임에 완전히 사라져 메타데이터에 `Object`로만 박힌다. 컨테이너가 뭘 주입할지 모르므로 `@Inject('TOKEN')` 명시적 토큰이 필요하다.

**Q. `@Global()` 모듈과 일반 모듈, 동작상 차이가 있나? (코드 스플리팅?)**
A. 없다. 둘 다 싱글톤 1개로 동작·성능 동일. 코드 스플리팅은 네트워크 전송이 있는 프론트의 관심사고, 서버는 코드를 다 메모리에 로드한다. 차이는 "전역 가시성(편의) vs 명시적 import(의존성 추적)"라는 트레이드오프뿐.

**Q. Guard가 Pipe보다 먼저 실행되는 이유는?**
A. 인증 안 된 요청에 검증·변환을 돌리는 건 낭비라서. "들어올 자격" → "입력 검증" 순.

**Q. Spring Security가 "서블릿 앞단"이라더니 "Filter다"라는 게 모순 아닌가?**
A. 모순 아님. Security = Filter이고 Filter = DispatcherServlet 앞단이니, "Security = 서블릿 디스패처 전단"으로 일관됨. Security는 앞단 필터(URL 인가)와 핸들러 직전 `@PreAuthorize`(메서드 인가) 두 군데에 걸쳐 있고, NestJS Guard는 핸들러 직전 한 군데서 둘을 통합한다.

**Q. 인증을 미들웨어/Interceptor로 하면 안 되나?**
A. Guard가 "차단(boolean)"이라는 단일 책임에 맞는 추상이다. `return false` 한 줄로 403까지 처리. Interceptor는 호출을 감싸는 AOP(가공/로깅/캐싱)용이지 차단용이 아니다.

**Q. 단일 스레드면 백엔드 처리가 제대로 되나? 인스턴스를 여러 개 붙이나?**
A. IO-bound엔 오히려 강하다(논블로킹으로 수천 연결을 한 스레드로). 약점은 CPU-bound. 멀티코어는 스레드 대신 프로세스 N개(PM2/cluster/K8s)로 확보한다.

**Q. 이벤트 루프가 코루틴 같은 개념인가?**
A. 맞다. `async/await`는 stackless coroutine이고 `await`가 yield(양보점)다. 가장 똑같은 건 Python asyncio. 단 Go goroutine/Kotlin coroutine은 멀티코어로 퍼지지만 JS는 단일 스레드에 갇혀 프로세스 다중화가 필요하다.

**Q. Node엔 HikariCP 같은 커넥션 풀이 없나?**
A. 있다. ORM/드라이버에 내장(`pg.Pool`, TypeORM/Prisma). 다만 풀이 프로세스마다 독립이라 "프로세스 수 × 풀 크기"만큼 커넥션이 늘어 DB max_connections를 초과할 수 있고, 그래서 PgBouncer를 앞에 두기도 한다.

## 주의사항 / 자주 하는 실수

- **`@Injectable()` 누락** — 의존성이 없어도 붙여야 메타데이터가 방출돼 주입 대상으로 인식됨
- **인터페이스 직접 주입** — 런타임에 사라짐. `@Inject('TOKEN')` 필요
- **`ValidationPipe`에 `whitelist:true` 누락** — over-posting(권한 상승) 취약점
- **`transform:true` 누락** — plain object라 class-validator가 데코레이터를 못 읽거나 타입 변환 안 됨
- **`@Global()` / `forwardRef` 남용** — 전자는 의존성 추적 불가, 후자는 설계 냄새(공유 로직 추출이 정답)
- **REQUEST 스코프 남용** — 성능 비용 + 상위 provider까지 스코프 오염
- **이벤트 루프 블로킹** — 동기 CPU 작업은 worker_threads/큐로 offload
- **커넥션 풀 곱셈 간과** — 프로세스 수 × 풀 크기 = 총 커넥션. PgBouncer 고려

## 참고

- [NestJS 공식 문서](https://docs.nestjs.com/)
- [백엔드 프레임워크 비교](백엔드-프레임워크-비교.md) ← Express/Koa/Fastify/Spring/FastAPI/Django
- [GraphQL 기본](GraphQL-기본.md)
- [폼 라이브러리 + 검증 비교](../폼-유효성검사/폼-라이브러리-검증-비교.md) ← class-validator
