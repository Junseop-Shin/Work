# 2장 — API 테스트

> 2026-06-04 | 모던 API 아키텍처, 테스트, ContractTesting, TestPyramid, Testcontainers, E2E

> 📖 [모던 API 아키텍처 — 책 목차/진행](README.md) · 이전: [1장 설계·REST·gRPC·OpenAPI](01-API설계-REST-gRPC-OpenAPI.md)

## 한 줄 요약

테스트는 **피라미드**대로(unit 많이, E2E 적게) 쌓고, MSA에선 **Contract Testing**이 E2E 의존을 줄이는 핵심이다(OpenAPI 명세 = 계약). Component(격리·mock)와 Integration(Stub/Testcontainers)을 구분하고 E2E는 최소화하며, 모든 전략 결정은 ADR로 남긴다.

## 1. 테스트 전략 — 피라미드 & 사분면

**Test Pyramid** (Mike Cohn):

```
        ▲  E2E          적음 / 느림 / 비쌈 / flaky
       ▲▲  Integration
      ▲▲▲  Component
     ▲▲▲▲  Unit          많음 / 빠름 / 쌈 / 안정적
```

- 아래일수록 많이·빠르게·싸게. 위일수록 느리고 취약하고 비쌈
- **안티패턴 = 아이스크림콘**(E2E만 잔뜩, unit 부족) → 느리고 깨지기 쉬운 슈트

**Test Quadrant** (Brian Marick): 2축(비즈니스 대면 ↔ 기술 대면 / 팀 지원 ↔ 제품 비평)으로 "무엇을 왜 테스트하나" 분류. API는 주로 **기술-팀지원**(unit/component/integration), **기술-비평**(성능/보안).

## 2. Contract Testing — 핵심

**문제:** MSA에서 A(consumer)가 B(producer) API에 의존. B가 API를 바꾸면 A가 깨지는데, E2E로 잡으면 느리고 늦게 발견.

**계약 테스트:** producer↔consumer 간 **계약(contract)**을 명시하고 **양쪽이 독립 검증**.

```
Consumer-Driven Contract (CDC, 예: Pact)
  consumer가 "이런 응답을 기대한다" 계약 정의
        ↓
  producer가 자기 CI에서 "계약 만족하나" 검증
        ↓
  계약 깨지면 배포 전 CI에서 잡힘 → 독립 배포 안전
```

**왜 선호:**
- producer/consumer **독립 배포** 안전 (전체 시스템 기동 불필요)
- **E2E보다 빠르고 안정적**

> **1장과 직결:** OpenAPI 명세가 곧 **계약**. design-first면 명세 자체가 contract이고, 계약 테스트로 준수를 강제. → [프론트-백엔드 계약 동기화](../../아키텍처/프론트-백엔드-계약-동기화.md)

## 3. Component Testing

단일 서비스(컴포넌트)를 **격리** 테스트. 외부 의존성(다른 서비스·DB)은 stub/mock으로 대체, 서비스 경계 안 동작을 통째로 검증.

| | Contract Testing | Component Testing |
|---|---|---|
| 검증 대상 | **인터페이스 합의** | **서비스 내부 행동** |
| 관계 | 보완 | 보완 |

> 어제 연결: **NestJS DI의 mock 주입**(`{ provide: Repo, useClass: MockRepo }`)이 격리 컴포넌트 테스트를 가능케 함. Data Mapper가 테스트 쉬운 이유와 동일. → [NestJS 아키텍처](../../백엔드/NestJS-아키텍처.md), [ORM 패턴 비교](../../ORM-ODM/ORM-패턴-비교.md)

## 4. Integration Testing — Stub Server & Testcontainers

서비스가 외부(DB·다른 API)와 **실제 통합**되는 지점 검증:

| 도구 | 역할 |
|------|------|
| **Stub Server** (WireMock 등) | 외부 API를 가짜 서버로 대체 → 외부 의존 없이 통합 로직 테스트 |
| **Testcontainers** | 실제 의존성(PostgreSQL·Kafka 등)을 **Docker 컨테이너**로 띄움 → mock보다 실제에 가깝고 격리·재현성↑ |

> Testcontainers는 **커넥션 풀/DB**(금요일 Redis/PostgreSQL)와 **Docker**에 연결 — "진짜 DB를 일회용 컨테이너로 띄워 통합 테스트".

## 5. End-to-End Testing

클라→게이트웨이→여러 서비스→DB 전체 흐름 검증. **느리고 flaky하고 유지비 높음** → 피라미드 상단답게 **최소화**, 자동화 필수. 종류(API 기반, UI 기반 등).

## 6. ADR 반복

각 전략(테스트 전략/계약/통합/E2E)마다 **ADR 가이드라인** 제시 — 0장 ADR을 실전에서 반복 적용하는 책의 일관된 패턴.

## 핵심 질의응답

**Q. MSA에서 E2E 테스트만으로 충분하지 않은 이유는?**
A. E2E는 느리고 flaky하며 전체 시스템 기동이 필요해 늦게 깨진다. Contract Testing으로 인터페이스 호환을 빠르게·독립적으로 검증하는 게 핵심이다.

**Q. Contract Testing과 Component Testing의 차이?**
A. 계약 테스트는 producer↔consumer "인터페이스 합의"를, 컴포넌트 테스트는 단일 서비스 "내부 행동"을 검증한다. 보완 관계.

**Q. Stub Server와 Testcontainers는 언제 쓰나?**
A. Stub는 외부 API를 가짜로 대체해 의존 없이 로직만 검증할 때, Testcontainers는 실제 DB/브로커를 Docker로 띄워 실제에 가까운 통합을 검증할 때.

**Q. OpenAPI 명세가 테스트와 무슨 관계?**
A. 명세가 곧 계약이라, design-first면 그 명세를 contract testing의 기준으로 삼아 producer/consumer 양쪽이 준수를 검증한다.

## 주의사항 / 자주 하는 실수

- **아이스크림콘 안티패턴** — E2E 과다, unit 부족 → 느리고 취약
- **E2E로 모든 걸 잡으려 함** — contract/component로 내려야 빠르고 안정적
- **mock 과신** — 외부 실제 동작과 다를 수 있음. 중요 통합은 Testcontainers로 실물 검증
- **계약 없이 독립 배포** — producer 변경이 consumer를 조용히 깸

## 참고

- [Mastering API Architecture — O'Reilly Ch.2 Testing APIs](https://www.oreilly.com/library/view/mastering-api-architecture/9781492090625/ch02.html)
- [Pact (Contract Testing)](https://pact.io/) · [Testcontainers](https://testcontainers.com/)
- [1장 설계·REST·gRPC·OpenAPI](01-API설계-REST-gRPC-OpenAPI.md) ← OpenAPI 명세=계약
- [프론트-백엔드 계약 동기화](../../아키텍처/프론트-백엔드-계약-동기화.md) ← contract 기반 동기화
- [NestJS 아키텍처](../../백엔드/NestJS-아키텍처.md) ← DI mock으로 격리 테스트
