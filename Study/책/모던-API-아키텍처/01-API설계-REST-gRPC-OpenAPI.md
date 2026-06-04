# 1장 — 설계·구현·명세 / REST / gRPC / OpenAPI

> 2026-06-04 | 모던 API 아키텍처, REST, gRPC, OpenAPI, 버저닝, API게이트웨이, 마셜링

> 📖 [모던 API 아키텍처 — 책 목차/진행](README.md) · 이전: [0장 도입](00-도입-사전개념.md)

## 한 줄 요약

1장은 "빠르게 만들 순 있지만 **오래가는(durable) API**는 표준·명세·버저닝·형식 선택을 신중히 하고 그 결정을 ADR로 남겨라"가 관통 메시지다. API는 설계→명세(OpenAPI)→구현 3계층이며, REST(리소스)·gRPC(고성능 내부)·GraphQL(유연 쿼리)을 트래픽·성능·클라이언트에 맞춰 고른다. (책 케이스 스터디: Attendee API)

## 1. API 설계 vs 구현 vs 명세

| | 무엇 | 산출물 |
|---|------|--------|
| **설계(Design)** | API가 **무엇을** 노출/**어떻게** 생겼나 | 결정 |
| **명세(Specification)** | 설계를 **형식화한 계약**(기계가 읽음) | OpenAPI/스키마 |
| **구현(Implementation)** | 명세를 **코드로** | 서버 코드 |

```
Design-first(Contract-first): 명세 먼저 → 합의 → 코드 생성/구현   ← 책이 미는 방향
Code-first:                   코드 먼저 → 명세 자동 생성          ← FastAPI/NestJS 방식
```

> 명세 = 설계와 구현 사이의 계약. design-first면 프론트·백이 명세에 합의하고 **병렬** 작업 가능.

## 2. REST의 5가지 필수 제약

| # | 제약 | 의미 |
|---|------|------|
| 1 | Client-Server | 관심사 분리(UI ↔ 데이터/로직) |
| 2 | **Stateless** | 서버가 세션 상태 안 가짐, 요청 자기완결 |
| 3 | Cacheable | 캐시 가능 여부 명시 |
| 4 | **Uniform Interface** | 균일 인터페이스 (REST의 정수) |
| 5 | Layered System | 계층화(클라는 중간 계층 모름) |
| (6) | ~~Code-on-Demand~~ | 선택 |

**Uniform Interface의 4하위 제약:** 리소스 식별(URI) / 표현을 통한 조작 / self-descriptive 메시지 / **HATEOAS**(하이퍼미디어).

> 자세한 stateless·REST 제약·세션 논의 → [인증 종합](../../인증/인증-JWT-OAuth-OIDC.md)

## 3. 리차드슨 성숙도 모델 (RMM)

| Level | 내용 | 현실 |
|-------|------|------|
| 0 | HTTP를 RPC처럼(단일 엔드포인트 POST) | 레거시 |
| 1 | 리소스별 URI 분리 | |
| 2 | HTTP 메서드 + 상태코드 제대로 | ← **대부분 여기** |
| 3 | HATEOAS(응답에 다음 액션 링크) | 진짜 REST, 극소수 |

## 4. 교환 방식 — REST / gRPC / GraphQL

| | REST | gRPC | GraphQL |
|---|------|------|---------|
| 중심 | 리소스(명사) | 액션(동사)=함수 호출 | 쿼리 |
| 형식 | JSON/HTTP | **Protobuf/HTTP2**(바이너리) | JSON/HTTP |
| 강점 | 표준·캐싱·범용 | 성능·스트리밍·강타입 | 응답 모양 클라가 결정 |
| 적합 | 공개(수직) API | **내부 서비스 간(수평)** | 다양한 클라 |

### 마셜링(Marshalling) vs 직렬화(Serialization) 〔Gemini 보강〕

- **직렬화**: 객체 상태를 저장/전송 가능한 바이트 스트림으로 (상태 저장 초점)
- **마셜링**: 직렬화 + **통신 규약(protocol)에 맞춰 포장**까지. 주로 RPC/IPC. 포인터처럼 원격에서 못 쓰는 것을 값으로 변환하는 로직 포함 → 더 넓은 개념
- 실무에선 거의 혼용. gRPC에서 객체→Protobuf 바이너리로 펴는 게 마셜링, 복원이 언마셜링.

### gRPC가 빠른 이유

1. **HTTP/2**: 한 커넥션에 여러 요청(멀티플렉싱) + 헤더 압축
2. **Protocol Buffers**: 텍스트(JSON) 대신 바이너리. **필드명을 빼고**(`.proto`로 사전 합의한 번호로) 전송 → 용량↓, 파싱 비용↓. IoT가 대역폭·전력 아끼려 바이너리 쓰는 철학과 동일

## 5. REST API 표준 구조 — 컬렉션 / 페이징 / 필터 / 에러

**컬렉션:** `/attendees`(컬렉션) ↔ `/attendees/123`(멤버), 복수형 명사.

**페이징:**

| 방식 | 예 | 특징 |
|------|-----|------|
| Offset | `?offset=20&limit=10` | 간단·점프 가능 / 큰 offset 느림, 데이터 변하면 중복·누락 |
| Cursor | `?cursor=...&limit=10` | 실시간에 안정적 / 임의 점프 불가 |

**필터/정렬:** `?status=active&sort=-createdAt`

**에러:** 표준 **RFC 7807 (Problem Details)** — `type/title/status/detail/instance`. HTTP 상태코드(→ [인증: 401/403/404](../../인증/인증-JWT-OAuth-OIDC.md)) + 일관된 에러 바디.

## 6. OpenAPI — 명세의 실전 가치

| 활용 | 내용 |
|------|------|
| **Code Generation** | 명세 → 서버 스텁·클라 SDK 생성 |
| **Validation** | 요청/응답이 명세 준수하는지 검증 |
| **Examples & Mocking** | 명세로 mock 서버 → 프론트 병렬 개발 |
| **Detecting Changes** | breaking change 탐지(CI) |

> design-first의 위력 = 명세가 단일 진실 → 코드생성·검증·mock·변경감지가 따라옴. → [프론트-백엔드 계약 동기화](../../아키텍처/프론트-백엔드-계약-동기화.md)

## 7. 버저닝 — 시맨틱 버저닝

**SemVer (MAJOR.MINOR.PATCH):** MAJOR=breaking, MINOR=하위호환 기능추가, PATCH=하위호환 버그수정.

| 위치 | 예 |
|------|-----|
| URL(흔함·명시적) | `/v1/attendees`, `/v2/attendees` |
| 헤더 | `Accept: application/vnd.api.v2+json` |
| 쿼리 | `?version=2` |

> breaking change를 피하고(additive 선호), 불가피하면 MAJOR↑ + deprecation 정책으로 구버전 유지.

## 8. 교환 형식 선택 — 트래픽 / 대용량 / HTTP

### API 게이트웨이 패턴 — 외부 HTTP, 내부 gRPC 〔Gemini 보강〕

```
[웹/앱] ──HTTP/JSON──► [API 게이트웨이] ──gRPC──► [주문]◄►[결제]◄►[재고]
         (범용·브라우저)   (인증/라우팅/변환)        (내부 수평, 고속 바이너리)
```

- **외부(수직)에 HTTP/REST:** 브라우저가 gRPC를 온전히 못 다룸(grpc-web 복잡), JSON은 범용 공용어
- **내부(수평)에 gRPC:** 서비스 간 초당 수만 호출 → 바이너리+HTTP2로 레이턴시·CPU 절감
- "가게 앞마당(외부)=HTTP, 주방 안쪽(내부)=gRPC 전용 무전기"

### 게이트웨이의 gRPC 처리 2방식

| 방식 | 동작 | 비고 |
|------|------|------|
| **Pass-Through** | HTTP/2를 그대로 통과(내용 안 봄) | 설정 쉬움 (Spring Cloud Gateway, ALB) |
| **Transcoding** | 외부 HTTP/JSON → 내부 gRPC로 변환 | Envoy/Kong/GCP가 강함 |

- **매니지드 게이트웨이는 "코드"가 아니라 `.proto` 디스크립터(규격서)를 등록** → 게이트웨이 엔진이 HTTP↔gRPC 매핑·마셜링 자동 처리. 비즈니스 로직 코드는 안 들어감
- **성능 함정:** 게이트웨이 구간만 보면 **Pass-Through가 더 빠름**(Transcoding은 JSON↔바이너리 변환 CPU 비용). gRPC 변환의 이득은 **내부 연쇄 호출(chaining)**에서 나옴 — 한 번 바이너리로 바꿔두면 뒤 서비스들이 JSON 파싱 없이 통신. **뒷단 서버 1대면 Pass-Through, 수십 대 MSA면 Transcoding이 전체 이득**

### 수직 vs 수평 속도 스케일 〔Gemini 보강 — ⚠️ 환경 의존, 감각용〕

| | 단일 요청 | 서버 1대당 RPS |
|---|----------|---------------|
| HTTP/JSON(수직) | ~10–50ms | ~1.5k–2.5k |
| gRPC(수평) | ~1–2ms | ~15k–25k |

> ⚠️ **"약 10배"는 직렬화·네트워크가 병목일 때의 감각치.** DB·비즈니스 로직이 병목이면 차이가 크게 줄어든다(어제 [비교 문서](../../백엔드/백엔드-프레임워크-비교.md): "벤치마크는 환경 의존, IO-bound면 의미 축소"). 절대 수치로 신봉 X.

### HTTP/2 → HTTP/3 〔Gemini 보강〕

- **HTTP/2**: 멀티플렉싱(한 커넥션 여러 요청), 헤더 압축. 단 TCP라 한 패킷 유실 시 전체 지연(HOLB)
- **HTTP/3**: 이미 표준(**RFC 9114**), 구글/넷플릭스 등 사용 중. **TCP 대신 UDP 기반 QUIC** → 핸드셰이크 단축, 스트림 독립(HOLB 해소), 모바일 IP 변경에도 연결 유지

### 대용량(엑셀 다운로드 등) 처리 〔Gemini 보강〕

외부 다운로드는 gRPC 아닌 **HTTP**(브라우저·파일 저장). 내부 대용량은 **gRPC Server Streaming**(청크 단위, OOM 방지). 실무 3정석:

1. **HTTP Response Stream** — DB를 한 줄씩 읽어 응답 스트림으로 (서버 메모리 안전 / 커넥션 오래 유지)
2. **비동기 배치 + 스토리지** ⭐ — 요청 접수만 응답 → 워커가 백그라운드로 파일 생성 → S3 업로드 → 알림/폴링으로 링크 (대용량 정석, → 금요일 큐/BullMQ)
3. **프론트 빌드** — 서버는 JSON만, 브라우저가 SheetJS로 엑셀 생성 (서버 부담 최소, 소규모)

## 9. MSA 도입 기준 + 모듈형 모놀리스 〔Gemini 보강〕

> "**첫 번째 규칙은 MSA를 하지 않는 것**"(마틴 파울러). 대부분 모놀리스 + 스케일업/아웃 + DB 튜닝/캐싱으로 충분.

**MSA 합리적 시점 (대략):** DAU 50만~100만+, Peak 5k~10k RPS+. **단 진짜 신호는 트래픽보다 ① 트래픽 비대칭(특정 기능만 폭증) ② 조직 규모(개발자 50명+ → 배포 병목)**.

**Spring Modulith (모듈형 모놀리스):** 코드는 도메인별로 MSA처럼 분리, 배포는 단일 프로세스.
- **성능 저하 거의 없음** — 모듈 간 통신이 네트워크가 아니라 **단일 JVM 내 메서드 호출**(나노초, 마셜링 0). 실제 MSA(네트워크+마셜링)보다 빠름
- 영향 요인: 이벤트 기반 통신(Event Publication Registry의 DB I/O), **공유 DB 병목**(모듈 분리해도 DB 하나면 JOIN/조회 패턴이 발목)
- 전략: 모듈형 모놀리스로 시작 → 트래픽·조직 커지면 그 모듈만 gRPC 서비스로 독립 (선MSA의 분산 트랜잭션 지옥 회피)

## 10. 다중 명세 (Multiple Specifications)

한 서비스가 REST + gRPC를 둘 다? **"Golden Specification(만능 단일 명세)은 없다."** 둘을 유지하면 중복·동기화·일관성 비용 → 한 소스에서 생성하거나 명확한 경계 필요, 결정은 ADR로.

## 핵심 질의응답 (책 + Gemini 학습)

**Q. 마셜링과 직렬화의 차이?**
A. 직렬화는 객체→바이트스트림(저장/전송 초점). 마셜링은 거기에 통신 규약 포장·포인터 값변환까지 포함하는 더 넓은 개념(RPC/IPC 맥락). 실무 혼용.

**Q. 외부는 HTTP, 내부 MSA는 gRPC가 표준인가?**
A. 그렇다(API 게이트웨이 패턴). 브라우저 호환·범용성 때문에 외부는 HTTP/REST, 내부는 성능 위해 gRPC.

**Q. 매니지드 게이트웨이에 gRPC 변환 코드를 심나?**
A. 아니다. `.proto` 디스크립터(규격서)를 등록하면 게이트웨이 엔진이 HTTP↔gRPC 매핑·마셜링을 자동 처리. 비즈니스 로직 코드는 안 들어간다.

**Q. Pass-Through와 Transcoding 중 뭐가 빠른가?**
A. 게이트웨이 구간만 보면 Pass-Through(변환 비용 없음). 하지만 내부 연쇄 호출이 많은 MSA면 한 번 gRPC로 변환해두는 Transcoding이 전체 레이턴시에서 이득. 뒷단 1대면 Pass-Through.

**Q. 수직 HTTP vs 수평 gRPC, 10배 빠르다는 게 항상 맞나?**
A. 아니다. 직렬화·네트워크가 병목일 때의 감각치다. DB·비즈니스 로직이 병목이면 차이가 작아진다. 벤치마크는 환경 의존.

**Q. 트래픽이 어느 정도면 MSA가 합리적인가?**
A. 대략 DAU 50만~100만, 5k~10k RPS 이상이지만, 절대 공식은 없다. 진짜 신호는 트래픽 비대칭성과 조직 규모(배포 병목)다. 그 전엔 모놀리스+스케일업이 정답.

**Q. Spring Modulith로 모듈 분리하면 성능 저하 없나?**
A. 거의 없다. 단일 JVM 내 메서드 호출이라 네트워크·마셜링 비용이 0. 영향은 이벤트 기반 통신의 DB I/O와 공유 DB 병목에서 온다. 실제 MSA보다 빠르다.

**Q. HTTP/3은 출시됐나? 웹3와 같은 건가?**
A. HTTP/3은 표준(RFC 9114) 완료·사용 중, UDP 기반 QUIC. 웹3(블록체인 탈중앙 패러다임)와는 이름만 비슷한 완전 별개.

**Q. 대용량 엑셀 다운로드는 gRPC/스트리밍?**
A. 외부 다운로드는 HTTP. 내부 대용량 조회는 gRPC server streaming. 실무는 HTTP 스트림 / 비동기 배치+S3(정석) / 프론트 SheetJS 중 선택.

## 주의사항 / 자주 하는 실수

- **REST 성숙도 Level 2를 REST 전부로 착각** — HATEOAS(Lv3)는 별개
- **gRPC = 무조건 빠름** — 게이트웨이 변환 구간은 오히려 느릴 수 있음, 이득은 내부 체이닝에서
- **RPS 벤치마크 절대 신봉** — 환경·병목 위치 의존
- **선(先) MSA** — 마틴 파울러 "MSA 하지 마라"가 첫 규칙. 모듈형 모놀리스로 시작
- **모듈형 모놀리스인데 공유 DB를 막 JOIN** — 모듈화 의미 퇴색 + 추후 분리 어려움
- **Golden Specification 추구** — 만능 단일 명세는 없다

## 참고

- [Mastering API Architecture — O'Reilly Ch.1](https://www.oreilly.com/library/view/mastering-api-architecture/9781492090625/ch01.html)
- [HTTP/3 (RFC 9114)](https://www.rfc-editor.org/rfc/rfc9114.html) · [C4 모델](https://c4model.com/)
- [0장 도입](00-도입-사전개념.md) ← C4·ADR·수직/수평 트래픽
- [인증 종합](../../인증/인증-JWT-OAuth-OIDC.md) ← REST stateless, 트래픽별 인증(수직 OAuth/수평 mTLS)
- [백엔드 프레임워크 비교](../../백엔드/백엔드-프레임워크-비교.md) ← gRPC/REST/GraphQL, MSA, 성능
- [GraphQL 기본](../../백엔드/GraphQL-기본.md)
