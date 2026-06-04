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

## 7. 실무 — 테스트 범위 / TDD / BDD 문서화

(책 본문 밖, 개발자 실무 관점 정리)

### 테스트를 어디까지 만드나 — risk-based

전부 X. **위험 기반**으로 깊이를 정한다.

| 영역 | 깊이 |
|------|------|
| 결제·인증·데이터 정합성 (실패=돈/보안) | 깊게 (unit+component+edge+일부 E2E) |
| 핵심 happy path | 반드시 (E2E 1개 + unit) |
| 에러·경계 경로 | 위험도 따라 (unit/component) |
| 부가·단순 표시 | 얕게 or 수동 |

- **범위 기준 = PO와 합의한 Acceptance Criteria.** 모호하면 PO에 명확화 요청
- 피라미드: **유스케이스 = E2E happy path 소수**, 그 안의 **로직·분기·에러 = unit/component 다수**. "유스케이스마다 E2E"는 아이스크림콘 안티패턴

### TDD — Double-loop

```
바깥 루프: Acceptance Test (요구사항/유스케이스, 기획 기반)
   └ 안쪽 루프: Unit Red-Green-Refactor (개발자가 세부 분해, 빠르게 다수)
   안쪽 여러 번 → 바깥 통과 → 기능 완성
```

- 기획 = **what**, 개발자 = **how + 엣지케이스**(빈 입력·경계값·동시성·실패 경로)
- 기술적 엣지는 개발자가 정의, **모호한 비즈니스 규칙은 PO 확인**

### 커버리지 — 목표가 아니라 참고

- **TDD는 "얼마나 테스트할지"를 자동으로 안 정해준다.** "무엇을 테스트할지(Red 목록)"는 인수조건 + risk가 결정
- 커버리지 숫자(line/branch) = **"테스트 안 한 곳 찾기"용 참고 지표**. 목표 X
- **100% ≠ 충분** (엣지케이스·허술한 단언이 빠질 수 있음). "위험 경로·핵심 규칙·인수조건을 다 검증했나"가 진짜 기준

### 테스트 = 살아있는 명세 (executable specification)

Red를 쓰는 행위 = "이 동작이 done이다"를 명세화(문서화). 한 번 쓴 테스트가 **3역할:**

```
① 명세 (짤 때)      ② 검수 기준 (Green=완료)      ③ 회귀 안전망 (이후 깨지면 빨강)
```

- 일반 문서(Word/Confluence)와 달리 **코드와 항상 동기화** — 어기면 빨강이라 "거짓말 못 함"
- 단 테스트가 대체하는 건 **"상세 동작 스펙"**이지, 비즈니스 요구사항·기획(why/배경)은 여전히 별도 문서

### BDD / Cucumber — 기획 + 테스트 + 문서 통합

테스트 케이스 = 기획 요구사항 = 문서를 **하나로** 묶는 방식.

```gherkin
Feature: 포인트 결제            ← 기획 설명(자연어)
  Scenario: 잔액 부족
    Given 사용자 포인트가 500이고
    When 800 결제를 시도하면
    Then 결제는 거부되고 "포인트 부족" 메시지가 표시된다
```

- `.feature`(Gherkin)는 **자연어** → 기획자도 읽고 씀(코드 불필요). step definition 코드만 개발자
- **3 amigos**(기획·개발·QA)가 함께 시나리오 작성 → 요구사항 합의 = 테스트 = 문서
- 도구: **Cucumber**(다언어), SpecFlow(.NET), Behave(Python), **Serenity BDD**(리포트+요구사항 추적)
- ※ API 레퍼런스 자동화는 별개: **OpenAPI/Swagger**(코드 기반, 불일치 가능), **Spring REST Docs**(테스트 통과해야 문서 생성 → 실제 동작 보장)

## 핵심 질의응답

**Q. MSA에서 E2E 테스트만으로 충분하지 않은 이유는?**
A. E2E는 느리고 flaky하며 전체 시스템 기동이 필요해 늦게 깨진다. Contract Testing으로 인터페이스 호환을 빠르게·독립적으로 검증하는 게 핵심이다.

**Q. Contract Testing과 Component Testing의 차이?**
A. 계약 테스트는 producer↔consumer "인터페이스 합의"를, 컴포넌트 테스트는 단일 서비스 "내부 행동"을 검증한다. 보완 관계.

**Q. Stub Server와 Testcontainers는 언제 쓰나?**
A. Stub는 외부 API를 가짜로 대체해 의존 없이 로직만 검증할 때, Testcontainers는 실제 DB/브로커를 Docker로 띄워 실제에 가까운 통합을 검증할 때.

**Q. OpenAPI 명세가 테스트와 무슨 관계?**
A. 명세가 곧 계약이라, design-first면 그 명세를 contract testing의 기준으로 삼아 producer/consumer 양쪽이 준수를 검증한다.

**Q. 요구사항을 어디까지 받아서 테스트를 어디까지 만드나?**
A. 전부가 아니라 risk-based. 실패 비용 큰 곳(결제·인증)을 깊게, 저위험은 얕게. 범위 기준은 PO와 합의한 Acceptance Criteria. 유스케이스는 E2E happy path 소수, 내부 로직·분기·에러는 unit/component 다수(피라미드).

**Q. TDD에서 큰 기획 요구사항과 작은 unit 테스트를 어떻게 잇나?**
A. Double-loop. 바깥 루프는 Acceptance(요구사항/유스케이스), 안쪽 루프는 Unit Red-Green-Refactor(개발자가 세부·엣지를 분해). 기획=what, 개발자=how+엣지.

**Q. 커버리지는 몇 %가 적당한가?**
A. 숫자는 목표가 아니라 "안 한 곳 찾기"용 참고다. 100%여도 엣지·허술한 단언이 빠질 수 있다. TDD가 범위를 자동으로 정해주지 않으므로, 무엇을 테스트할지는 인수조건+risk로 정한다.

**Q. 테스트가 문서를 대체하나? 기획자가 코드를 읽어야 하나?**
A. 테스트는 "상세 동작 스펙"을 대체하는 살아있는 명세지만, 비즈니스 요구사항·기획(why)은 여전히 별도 문서다. BDD(Gherkin)는 자연어라 기획자도 읽고 쓴다(코드 불필요) — unit만 개발자 영역.

**Q. 테스트 케이스를 기획 요구사항과 함께 정리하는 문서화 도구는?**
A. BDD의 Cucumber(Gherkin)다. `.feature` 파일이 기획 요구사항 + 테스트 케이스 + living documentation을 하나로 묶는다. SpecFlow/Behave/Serenity도 같은 범주. (API 레퍼런스 자동화인 Swagger/Spring REST Docs와는 별개)

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
