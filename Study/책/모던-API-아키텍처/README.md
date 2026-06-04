# 모던 API 아키텍처 (Modern API Architecture)

> 책 학습 정리 — 장마다 문서 하나씩. 필요 시 장을 묶거나 분리.

## 책 정보

| | |
|---|---|
| 제목 | 모던 API 아키텍처 |
| 분류 | API 설계 / REST / 아키텍처 |
| 학습 트랙 | Phase 2 / Week 5 — Track C |

*(저자·원제 등 메타는 추후 보완)*

## 장별 정리 현황

| 장 | 주제 | 문서 | 상태 |
|----|------|------|------|
| 0 | 도입 / 사전 개념 (API 정의·C4·트래픽·ADR) | [00-도입-사전개념](00-도입-사전개념.md) | ✅ |
| 1 | 설계·구현·명세 / REST / gRPC / OpenAPI (+게이트웨이·MSA·HTTP3 심화) | [01-API설계-REST-gRPC-OpenAPI](01-API설계-REST-gRPC-OpenAPI.md) | ✅ |
| 2 | API 테스트 (피라미드·Contract·Component·Testcontainers·E2E) | [02-API테스트](02-API테스트.md) | ✅ |
| 3+ | (이후 진행) | — | ⬜ |

## 정리 규칙

- 파일명: `NN-장제목.md` (번호 prefix로 순서 고정)
- 장마다 문서 1개 기본, 내용이 짧거나 이어지면 묶기도
- 재사용성 높은 핵심 개념(REST 원칙 등)은 `Study/아키텍처/` 등 주제 카테고리로 승격하고 상호 링크
  (책 노트 = 독서 흐름 / 카테고리 = 주제 검색)

## 관련 문서

- [인증 종합](../../인증/인증-JWT-OAuth-OIDC.md) — REST stateless, 401/403, 트래픽별 인증
- [백엔드 프레임워크 비교](../../백엔드/백엔드-프레임워크-비교.md) — REST 성숙도, MSA
- [GraphQL 기본](../../백엔드/GraphQL-기본.md) — REST vs GraphQL
