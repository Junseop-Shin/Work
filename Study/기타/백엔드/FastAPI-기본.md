# FastAPI 기본 — 타입 힌트 = 계약

> 2026-06-04 | FastAPI, Pydantic, ASGI, Depends, async, Starlette, NestJS비교

## 한 줄 요약

FastAPI는 **Python 타입 힌트를 1급으로 활용해, 단 하나의 타입 선언에서 검증·직렬화·문서(OpenAPI)·자동완성을 전부 파생**시키는 ASGI(비동기) 프레임워크다. "Starlette(웹) + Pydantic(데이터)"의 결합이며, TS와 달리 Python 타입 힌트는 런타임에 살아있어 메타데이터 트릭(NestJS의 `emitDecoratorMetadata`)이 필요 없다. NestJS가 관심사별 전용 추상(Guard/Pipe/Interceptor)을 주는 반면, FastAPI는 `Depends()` + 타입힌트로 통합한다.

> NestJS와의 구조 비교는 → [NestJS 아키텍처](NestJS-아키텍처.md), 언어 선택은 → [백엔드 프레임워크 비교](백엔드-프레임워크-비교.md)

## 핵심 개념

### 왜 FastAPI를 쓰는가 — Django/Flask의 빈틈

| | Django | Flask | → FastAPI |
|---|--------|-------|-----------|
| 성격 | 풀스택·배터리 포함 | 마이크로 | 모던 API |
| 동시성 | **WSGI(동기)** | **WSGI(동기)** | **ASGI(비동기)** |
| 타입 힌트 | 거의 안 씀 | 거의 안 씀 | **1급 활용** |
| 검증 | Forms/DRF | 수동(marshmallow) | **Pydantic 자동** |
| API 문서 | 수동/플러그인 | 수동 | **OpenAPI 자동** |

Django/Flask의 공통 한계 = WSGI(동기, WebSocket 불가) + 타입 힌트 미활용(검증·문서·직렬화 전부 수동/중복). FastAPI는 이 빈틈을 파고들었다.

### "타입 힌트 = 단일 진실 공급원"

```python
@app.post("/users")
async def create_user(user: CreateUser) -> UserResponse:
    return await save(user)
```

이 한 줄에서 FastAPI가 자동 생성: 요청 body 검증 + JSON 파싱 + 응답 직렬화/필터링 + `/docs`(Swagger)·`/redoc` + IDE 자동완성. NestJS가 DTO+`@IsEmail()`+`@nestjs/swagger`를 **따로** 다는 반면, FastAPI는 **타입 힌트 그 자체**가 모든 것의 출발점 → 중복 선언 없음.

### Python 타입 힌트는 런타임에 살아있다 (vs TS 소거)

어제 NestJS의 핵심이 "TS는 타입 소거 → `emitDecoratorMetadata`로 복원"이었다. **Python은 정반대 — 타입 힌트가 런타임에 보존**된다.

```python
def create_user(user: CreateUser): ...
create_user.__annotations__   # → {'user': <class 'CreateUser'>}  런타임에 직접 읽힘!
```

| | TypeScript (NestJS) | Python (FastAPI) |
|---|---|---|
| 런타임 타입 | 소거 → reflect-metadata로 복원 | **introspection으로 직접 접근** |
| 트릭 필요? | 필요 | **불필요** |

→ FastAPI는 함수 시그니처를 직접 introspect해 검증·문서를 만든다. 단 타입 힌트는 "힌트"라 **자동 강제는 안 됨** — FastAPI/Pydantic이 명시적으로 읽어 검증하는 것.

### 정체성 — "Starlette + Pydantic"

```
┌──────────── FastAPI ────────────┐  ← 타입힌트로 둘을 엮는 접착제 + DI + OpenAPI
│  Starlette (웹/ASGI)  │  Pydantic (데이터)  │
│  라우팅/미들웨어/WS    │  검증/직렬화/스키마   │
└──────────────────────┴────────────────────┘
              ASGI (uvicorn)
```

- Starlette = 경량 ASGI 툴킷 (NestJS의 Express/Fastify 어댑터 자리)
- Pydantic = 검증·직렬화 (NestJS의 class-validator + class-transformer 자리)

---

## Pydantic

### `BaseModel` 상속이 왜 필요한가 — 메타클래스

```python
class User(BaseModel):
    name: str
    age: int
```

`class User(BaseModel)` 정의 **시점**에 메타클래스(`ModelMetaclass`)가: ① 어노테이션을 읽어 ② 각 필드의 core schema를 빌드 → ③ `pydantic-core`(Rust)에 넘겨 컴파일된 검증기/직렬화기를 만들어 ④ `__pydantic_validator__`/`__pydantic_serializer__`로 붙인다. **그래서 상속 필수**(is-a 관계 — "User는 검증 가능한 모델이다"). 스키마는 정의 시 단 한 번 빌드 → 요청마다 재파싱 X(Fastify 스키마 사전 컴파일과 같은 발상).

### v2는 왜 Rust(`pydantic-core`)로 다시 쓰였나

> 공식: "V2부터 일부를 `pydantic-core`라는 별도 패키지에서 Rust로 작성. 검증·직렬화 성능 개선 목적(내부 로직 커스터마이징·확장성 제한을 비용으로)."

```
pydantic (Python)      → 모델 정의, 유연성
pydantic-core (Rust)   → 검증·직렬화 실행, 성능 (v1 대비 수~수십 배)
```

FastAPI는 **요청마다 검증**하므로 이 속도가 처리량에 직결. 단 IO-bound(DB 병목)에선 전체 처리량 향상은 제한적 — 검증이 무거운 경우(거대/중첩 페이로드)에 체감.

### 검증 — lax(기본) vs strict

```python
class M(BaseModel):
    x: int

M.model_validate({'x': '123'})              # lax(기본): '123' → 123 (강제 변환)
M.model_validate({'x': '123'}, strict=True) # strict: ValidationError
M.model_validate_json('{"x": 123}')         # JSON을 Rust에서 직접 파싱+검증 (더 빠름)
```

### 직렬화 + v1→v2 API 변경

```python
user.model_dump()                            # → dict
user.model_dump_json()                       # → JSON 문자열
user.model_dump(exclude={'password'}, exclude_none=True)
```

| v1 | v2 | | v1 | v2 |
|----|----|--|----|----|
| `.dict()` | `.model_dump()` | | `@validator` | `@field_validator` |
| `.json()` | `.model_dump_json()` | | `@root_validator` | `@model_validator` |
| `.parse_obj()` | `.model_validate()` | | `class Config` | `model_config = ConfigDict()` |

### class-validator vs Zod vs Pydantic — 검증의 "방향"

| | Pydantic | class-validator | Zod |
|---|---|---|---|
| 단일 소스 | **타입 힌트** | 클래스+데코레이터 | **스키마 코드** |
| 방향 | **타입 → 검증** | 타입+규칙 분리 | **스키마 → 타입(`z.infer`)** |
| 직렬화 | 통합 | 별도(class-transformer) | parse가 겸함 |

핵심: **어제 본 "TS 타입 소거"가 또 설계를 가른다.** Python은 타입이 런타임에 있어 "타입→검증"(Pydantic). TS는 소거되니 → class-validator(데코레이터+reflect-metadata로 복원) 또는 **Zod(스키마를 런타임 값으로 정의하고 타입을 역추론)**. Zod의 거꾸로 설계는 TS 타입 소거의 산물.

---

## Depends / 의존성 주입

### `Depends()`가 곧 FastAPI의 DI다 (컨테이너가 없을 뿐)

FastAPI는 DI를 **제공한다** — 간판 기능이 `Depends()`다. NestJS/Spring과 차이는 "DI 유무"가 아니라 **컨테이너 기반(자동 와이어링) vs 함수 기반(`Depends` 수동 체인)**.

```python
def get_repo(db: Session = Depends(get_db)) -> UserRepository:
    return UserRepository(db)
def get_service(repo: UserRepository = Depends(get_repo)) -> UserService:
    return UserService(repo)

@router.get("/users/{id}")
async def get_user(id: int, svc: UserService = Depends(get_service)):
    return svc.get(id)
```

### 계층(컨트롤러/서비스/레포)은 상속이 아니라 구성

FastAPI는 계층을 **강제하지 않지만** 관례로 만들 수 있다(`routers/`/`services/`/`repositories/`). **계층화에 상속은 불필요** — NestJS조차 서비스/레포는 `@Injectable`+생성자 주입(구성, has-a)이지 상속이 아니다. 상속(is-a)이 맞는 건 Pydantic `BaseModel`뿐. FastAPI 서비스/레포는 일반 클래스 + `Depends` 와이어링.

| | NestJS/Spring | FastAPI |
|---|---|---|
| 와이어링 | 컨테이너 **자동** | `Depends()` **수동** |
| 보일러플레이트 | 적음 | 많음(`get_xxx` 함수들) |
| 명시성 | 낮음(마법) | 높음(흐름이 코드에) |
| 테스트 교체 | `{provide, useClass}` | `app.dependency_overrides[get_service] = mock` |

### NestJS 추상 → FastAPI 매핑

| NestJS | FastAPI |
|--------|---------|
| Provider DI | `Depends()` |
| **Guard**(인증/인가) | `Depends(get_current_user)` (실패 시 `raise HTTPException(401)` → 핸들러 차단) |
| **Pipe**(검증) | Pydantic 타입 힌트 (자동) |
| **Exception Filter** | `@app.exception_handler` + `HTTPException` |
| Middleware | `@app.middleware("http")` |
| **Interceptor**(AOP) | ⚠️ 1급 대응 없음 (middleware / `yield` dependency로 부분) |

### 인증 — Guard를 Depends로

```python
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    user = decode_jwt(token)
    if not user:
        raise HTTPException(status_code=401)   # 차단 → 핸들러 실행 안 됨 (= Guard false)
    return user

@router.get("/profile")
async def profile(user: User = Depends(get_current_user)): ...  # Guard 역할
```

### 전역 인증 + 일부 공개 제외

```python
# 패턴 1 (정석) — 라우터 분리
public = APIRouter()                                        # 인증 없음 (/login, /health)
protected = APIRouter(dependencies=[Depends(get_current_user)])  # 그룹 전체 인증

# 패턴 2 — 미들웨어 + 화이트리스트 (NestJS 전역 Guard + @Public 흉내)
@app.middleware("http")
async def auth(request, call_next):
    if request.url.path in PUBLIC_PATHS: return await call_next(request)
    # ... 토큰 검증 ...
```

NestJS는 `@Public()` + Reflector(메타데이터 읽어 스킵)로 우아하게, FastAPI는 **라우터 분리**가 정석(더 명시적).

### `yield` 의존성 — 전후 처리 + 리소스 정리

```python
def get_db():
    db = SessionLocal()
    try:
        yield db        # yield 전 = setup(before)
    finally:
        db.close()      # yield 후 = teardown(after) — Koa 양파/Interceptor의 결
```

### 예외처리 — 전역만 지원

`@app.exception_handler`는 **앱(전역) 레벨만**. NestJS의 컨트롤러/메서드 레벨 Filter는 없음 → 부분 처리는 핸들러 내 `try/except` 또는 `yield` dependency로.

---

## async / ASGI

### WSGI vs ASGI

| | WSGI | ASGI |
|---|------|------|
| 모델 | 동기, 요청↔응답 1:1 | 비동기, 이벤트 기반 |
| 시그니처 | `def app(environ, start_response)` | `async def app(scope, receive, send)` |
| 동시성 | 워커 다중화(블로킹) | 단일 스레드 이벤트 루프(논블로킹) |
| WebSocket/SSE | ❌ | ✅ |
| 서버 | gunicorn, uWSGI | **uvicorn**, hypercorn |
| 프레임워크 | Django(전통), Flask | **FastAPI**, Starlette |

ASGI = "Python판 이벤트 루프 모델", uvicorn이 Node의 이벤트 루프 역할. `async def`가 `await`에서 양보(stackless coroutine) — 어제 NestJS·asyncio와 동작 모델 동일.

### `async def` vs `def` 핸들러 + 분열 함정

```python
@app.get("/a")
async def h_async(): await asyncpg_query()    # 이벤트 루프에서 실행 (await 필수)

@app.get("/b")
def h_sync(): legacy_sync_db()                 # FastAPI가 threadpool로 빼서 실행 (루프 안 막음)
```

**함정:** `async def` 안에서 동기 블로킹(`time.sleep`, `requests`, `psycopg2`) → **이벤트 루프 전체 정지**(어제 CPU-bound 함정). Python은 동기/비동기 라이브러리가 **분열**돼 있어(`requests`↔`httpx`, `psycopg2`↔`asyncpg`) async 핸들러에선 반드시 async 버전을 써야 함. "async인데 동기 드라이버"가 FastAPI 성능 버그 1순위.

---

## 보안 — ORM/스키마 노출

자세한 ORM 위험은 → [ORM 패턴 비교](../ORM-ODM/ORM-패턴-비교.md). FastAPI 관점 요점:

- **엔티티 직접 직렬화 금지** → `response_model=UserResponse`로 노출 필드만. (ORM 엔티티 ≠ API 응답)
- **입력 모델 분리** → over-posting 방어 (DB 엔티티와 입력 Pydantic 모델 분리)
- **DB 에러 그대로 노출 금지** → `@app.exception_handler`로 일반화 (테이블·컬럼명 누출 방지)
- **`/docs`·`/openapi.json` 프로덕션 노출 주의:**

```python
is_prod = os.getenv("ENV") == "production"
app = FastAPI(
    docs_url=None if is_prod else "/docs",
    redoc_url=None if is_prod else "/redoc",
    openapi_url=None if is_prod else "/openapi.json",  # 이것도 꺼야 JSON 노출 차단
)
```

> 단 **숨김 ≠ 보안**(security through obscurity). Swagger 끄기는 심층 방어의 한 겹일 뿐, 본질은 API가 인증/인가/검증으로 단단한 것.

---

## 핵심 질의응답

**Q. FastAPI는 NestJS처럼 데코레이터/상속을 왜 거의 안 써도 되나?**
A. 데코레이터의 NestJS적 목적이 "타입 메타데이터를 남기는 것"인데, Python은 타입 힌트가 런타임에 살아있어 그게 공짜다. 그래서 함수+타입힌트라는 가벼운 형태로 충분. `@app.post`는 쓰지만 라우팅 등록용일 뿐 타입 캡처용이 아니다.

**Q. FastAPI엔 컨트롤러/서비스/레포 계층이 없나? 있으려면 상속해야 하나?**
A. 프레임워크가 강제 안 할 뿐 관례로 만들 수 있다. 계층화에 상속은 불필요 — 일반 클래스 + `Depends()` 구성으로 한다. 상속이 맞는 건 Pydantic `BaseModel`(is-a)뿐.

**Q. FastAPI는 DI를 제공 안 하나? 필터/인터셉터/인증은?**
A. DI는 `Depends()`로 제공(간판 기능, 컨테이너가 없을 뿐). 인증=Depends, 검증=Pydantic, 예외=exception_handler, 미들웨어 있음. 유일하게 약한 건 Interceptor의 AOP(응답 변형).

**Q. 전역 인증인데 일부만 공개하려면?**
A. 라우터 분리(`APIRouter(dependencies=[...])`)가 정석. 또는 미들웨어+화이트리스트(NestJS 전역 Guard+@Public 흉내).

**Q. v1→v2 Rust 전환으로 FastAPI 처리량이 크게 오르나?**
A. 제한적. 웹은 IO-bound(DB 병목)라 검증 속도 향상이 전체에 덜 영향. 검증이 무거운 경우에만 체감 + CPU 여유로 이벤트 루프 블로킹 완화 간접 효과.

**Q. Zod는 왜 "스키마 먼저, 타입 추론"인가?**
A. TS 타입이 런타임에 소거되기 때문. 그래서 스키마를 런타임 값으로 정의하고 타입을 역추론(`z.infer`). Python(Pydantic)은 타입이 살아있어 "타입→검증"이 가능.

## 주의사항 / 자주 하는 실수

- **`async def` + 동기 라이브러리** — 이벤트 루프 블로킹. `httpx`/`asyncpg` 등 async 버전 사용
- **ORM 엔티티 직접 반환** — 민감 필드 노출. `response_model`로 분리
- **`docs_url`만 끄고 `openapi_url` 방치** — JSON 스키마 그대로 노출
- **v1 API 혼용** — `.dict()`/`@validator` 등은 v2에서 deprecated
- **`Depends` 보일러플레이트 폭증** — 큰 앱이면 `dependency-injector` 같은 외부 컨테이너 고려
- **Interceptor식 AOP 기대** — FastAPI엔 1급 없음. middleware/yield로 우회

## 참고

- [FastAPI 공식 문서](https://fastapi.tiangolo.com/)
- [Pydantic 공식 문서](https://docs.pydantic.dev/)
- [NestJS 아키텍처](NestJS-아키텍처.md) ← DI/생명주기/동시성 비교
- [ORM 패턴 비교](../ORM-ODM/ORM-패턴-비교.md) ← Active Record vs Data Mapper
- [백엔드 프레임워크 비교](백엔드-프레임워크-비교.md) ← Python vs Node 선택
- [폼 라이브러리 + 검증 비교](../폼-유효성검사/폼-라이브러리-검증-비교.md) ← class-validator/Zod
