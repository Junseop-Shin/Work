# ORM 패턴 비교 — Active Record vs Data Mapper

> 2026-06-04 | ORM, ActiveRecord, DataMapper, SQLAlchemy, TypeORM, Prisma, Mongoose, UnitOfWork

## 한 줄 요약

모든 ORM은 **"영속성 로직(save/delete)이 어디 사는가"**로 갈린다. **Active Record**는 도메인 객체 자신이 `save()`를 가짐(직관적·CRUD에 최적, 도메인-DB 결합). **Data Mapper**는 별도 Repository가 저장을 담당하고 객체는 순수함(테스트·복잡한 도메인에 유리, 보일러플레이트↑). SQLAlchemy·JPA·NestJS Repository는 DM, Django·Mongoose는 AR, TypeORM은 둘 다, Prisma는 독자적(쿼리빌더+코드생성).

## 핵심 개념

### Active Record — 객체가 자기 영속성을 안다

도메인 객체 **자신**에 `save()`/`delete()`가 있다. 객체가 곧 테이블 행이고, 저장 방법도 스스로 안다.

```python
# Django ORM
user = User(name="Kim", age=30)
user.save()                          # 객체가 자기를 저장
User.objects.filter(age__gte=20)
```

```typescript
// TypeORM — Active Record 모드 (BaseEntity 상속)
@Entity()
class User extends BaseEntity {      // 상속이 핵심
  @Column() name: string;
}
const u = User.create({ name: "Kim" });
await u.save();                       // 객체에 영속성 메서드
```

- **장점:** 직관적, 코드 적음, 빠른 개발 (CRUD 앱 최적)
- **단점:** 도메인 객체가 DB를 앎 → 로직과 영속성 **결합**. 테스트에 DB 필요. 복잡한 도메인에서 비대

### Data Mapper — 객체는 영속성을 모른다

도메인 객체는 **순수**(POPO/POJO)하고, 별도 **Mapper/Repository**가 객체↔DB 변환을 담당.

```python
# SQLAlchemy — Session이 매퍼 역할
user = User(name="Kim", age=30)      # 순수 객체 (DB를 모름)
session.add(user)                     # Session(매퍼)이 저장
session.commit()
```

```typescript
// TypeORM — Data Mapper 모드 (Repository)
const repo = dataSource.getRepository(User);
const u = repo.create({ name: "Kim" });  // 순수 엔티티
await repo.save(u);                        // Repository가 저장
```

- **장점:** 객체가 **순수** → 관심사 분리, 테스트 쉬움(DB 없이 도메인 로직), 복잡한 도메인·DDD 유리
- **단점:** 레이어 하나 더(Repository), 보일러플레이트↑, 러닝커브↑

### 한눈 비교

| | Active Record | Data Mapper |
|---|---|---|
| 영속성 로직 | **객체 안** (`user.save()`) | **별도 Repository** (`repo.save(user)`) |
| 도메인 객체 | DB를 앎 (결합) | 순수 (분리) |
| 결합도 | 높음 | 낮음 |
| 적합 | CRUD·빠른 개발 | 복잡한 도메인·DDD·테스트 중시 |
| 보일러플레이트 | 적음 | 많음 |

### NestJS DI와의 연결 — DM이 테스트가 쉬운 이유

NestJS의 `@InjectRepository(User)`가 바로 **Data Mapper**다. DM이 "테스트 쉽다"는 건 어제 본 DI 덕분 — Repository를 인터페이스로 두고 `{ provide: UserRepo, useClass: MockUserRepo }`로 **mock을 갈아끼울 수 있어서** DB 없이 도메인 로직을 검증한다. AR은 객체가 DB와 묶여 있어 그게 어렵다. NestJS가 Repository(DM)를 미는 이유 = [NestJS](../백엔드/NestJS-아키텍처.md)의 계층 분리·테스트 철학과 일치하기 때문.

## 주요 ORM이 어디에 속하나

| ORM | 언어 | 패턴 |
|-----|------|------|
| **SQLAlchemy** | Python | **Data Mapper** (+ Unit of Work, Identity Map) |
| Django ORM | Python | Active Record |
| **TypeORM** | Node | **둘 다 지원** (AR / DM) — 큰 앱엔 DM 권장 |
| **Prisma** | Node | **독자적** (아래) |
| **Mongoose** | Node (MongoDB) | Active Record 성향 (`doc.save()`, 모델 메서드) |
| JPA/Hibernate | Java | Data Mapper (+ UoW) |

### Prisma — 어느 쪽도 아닌 독자 방식

```typescript
const user = await prisma.user.create({ data: { name: "Kim" } });
//                  ^^^^^^^^^^^ 생성된 타입 안전 클라이언트가 쿼리
```

- `schema.prisma`로 스키마 선언 → **타입 안전 클라이언트 코드 생성**
- 반환값은 **순수 객체**(메서드 없음), 모든 작업은 `prisma.user.xxx()` 클라이언트로
- Data Mapper에 가깝지만 Repository 패턴은 아닌 **"쿼리 빌더 + 코드 생성"** 하이브리드. "타입을 단일 소스로"(Zod·Pydantic) 철학의 ORM 버전

### Unit of Work (SQLAlchemy Session, Hibernate)

DM과 함께 오는 패턴. 변경된 객체들을 **추적**했다가 `commit()` 시 한 번에 DB 반영(트랜잭션 묶음 + 쿼리 최적화). `session.add()` 후 `commit()`이 그것. Active Record엔 보통 없는 개념. **Identity Map**(같은 행은 같은 객체 인스턴스로 보장)도 함께 온다.

## 보안 — ORM 엔티티를 경계 밖으로 흘리지 마라

ORM 자체는 보안을 약화하지 않는다. 오히려 **파라미터 바인딩으로 SQL Injection을 막아** 보안에 유리(raw query 섞을 때만 직접 방어). 진짜 위험은 **엔티티를 API 경계 밖으로 그대로 흘릴 때**:

- **엔티티 직접 직렬화** → `password_hash`/`is_admin` 노출 → 응답 전용 모델(`response_model`/응답 DTO)로 분리
- **over-posting** → 입력 DTO 분리 + `whitelist`(NestJS) / 입력 Pydantic 모델 분리
- **DB 에러 그대로 노출** → 테이블·컬럼명 누출 → Exception Filter/handler로 일반화
- **API 문서·introspection 프로덕션 노출**(`/docs`, GraphQL introspection) → 비활성화/인증 뒤

> "ORM이 DB 구조를 공개한다"는 오해다. 엔티티는 서버에만 있고, 서버 침해 시 DB 위험은 ORM 무관(자격증명 문제). 위험은 "엔티티를 직렬화·문서로 새어나가게 하는 것"이고, 방어책은 [NestJS](../백엔드/NestJS-아키텍처.md)·[FastAPI](../백엔드/FastAPI-기본.md)에서 본 DTO 분리·whitelist·Exception Filter.

## 핵심 질의응답

**Q. ORM을 가르는 가장 근본적인 기준은?**
A. "영속성 로직(save/delete)이 어디 사는가." 객체 안이면 Active Record, 별도 Repository면 Data Mapper.

**Q. Data Mapper가 테스트가 쉬운 이유는?**
A. 객체가 순수해 도메인 로직을 DB 없이 검증 가능 + Repository를 인터페이스로 두고 mock(`useClass: MockRepo`)으로 갈아끼울 수 있어서. AR은 객체가 DB와 묶여 어렵다.

**Q. Prisma는 Active Record인가 Data Mapper인가?**
A. 둘 다 아니다. `schema.prisma`로 스키마 선언 → 타입 안전 클라이언트 코드 생성 → `prisma.user.xxx()`로 쿼리. 반환은 순수 객체. "쿼리 빌더 + 코드 생성" 하이브리드.

**Q. ORM을 쓰면 DB 구조가 공개되는 보안 취약점이 있나?**
A. 아니다. ORM은 오히려 SQL Injection을 막아 보안에 유리. 진짜 위험은 ORM 엔티티를 API 응답으로 그대로 직렬화하거나 `/docs`를 프로덕션에 노출하는 것 → DTO 분리·문서 비활성화로 방어.

## 주의사항 / 자주 하는 실수

- **AR을 복잡한 도메인에** — 객체가 비대해지고 도메인-DB 결합이 발목. 복잡하면 DM
- **DM 보일러플레이트를 단순 CRUD에** — 과한 추상화. 간단하면 AR이 빠름
- **엔티티 = API 모델로 착각** — 노출 필드 분리 필수
- **TypeORM에서 AR/DM 혼용** — 한 프로젝트는 한 방식으로 통일 권장
- **Unit of Work 무지** — SQLAlchemy `session.add` 후 `commit` 안 하면 반영 안 됨

## 참고

- [SQLAlchemy](https://docs.sqlalchemy.org/) · [TypeORM](https://typeorm.io/) · [Prisma](https://www.prisma.io/docs)
- [NestJS 아키텍처](../백엔드/NestJS-아키텍처.md) ← @InjectRepository(DM), DI 테스트
- [FastAPI 기본](../백엔드/FastAPI-기본.md) ← SQLAlchemy + Pydantic 조합
