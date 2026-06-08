# Elasticsearch — 검색 엔진

> 2026-06-08 | Elasticsearch, 검색, 역색인, Lucene, BM25, 폴리글랏

## 한 줄 요약

Elasticsearch는 **역색인(빠른 조회) + 분석기(텍스트 정규화) + BM25(관련도 순위)** 를 결합해 "문자열 매칭"이 아니라 "검색"을 하는 엔진이다. 원본이 아니라 **PG/Mongo의 파생 색인**이며, 폴리글랏 아키텍처의 검색 계층을 맡는다.

## 핵심 개념

### 왜 ES — `LIKE`의 한계

RDB(`LIKE`)나 Mongo로도 텍스트 검색은 되지만 두 가지가 안 된다:

1. **인덱스** — `LIKE '검색어%'`(앞 고정)는 B-tree를 타지만, **`LIKE '%검색어%'`(중간 포함)는 못 타서 풀스캔.**
2. **"검색"이 아니라 "문자 그대로 매칭"** — `'%run%'`은 "running"·"ran"을 **못 찾고**(글자가 다름), **관련도 순 정렬이 없다**(맞다/아니다만).

ES는 이를 세 무기로 푼다:

| 무기 | 역할 | LIKE 대비 |
|------|------|-----------|
| **역색인** | 단어→문서목록 즉시 조회 | 풀스캔 해결 |
| **분석기** | "running"→"run" 정규화 | 글자 달라도 매칭 |
| **BM25** | 관련도 점수 | 순위 정렬 |

### 역색인 (Lucene)

문서를 단어로 쪼개(토큰화) **`단어 → 그 단어를 포함한 문서목록(posting list)`** 으로 뒤집어 저장:

```
문서1: "redis is fast"      "redis" → [문서1, 문서3]
문서3: "redis db"      →     "db"    → [문서2, 문서3]
```

"redis db" 검색 → `redis`목록 ∩ `db`목록 = **[문서3] 즉시** (풀스캔 0). → 오늘 PG의 **GIN(`값→문서목록`)과 같은 구조**고, **Lucene**이 이를 구현한 엔진이다. (ES = Lucene을 분산·REST로 감싼 것)

> ⚠️ **역색인만으론 ES만의 강점이 아니다** — PG도 전문검색(`tsvector`+GIN)이 된다. ES의 진짜 차별점은 그 위에 쌓인 **분석기 + BM25 + 분산/UX**.

### 분석기(analyzer) — 색인 "전에" 텍스트를 가공

역색인에 단어를 넣기 **전에** 3단계 파이프라인으로 가공한다:

```
"The cats are Running!"
  ① Character filter → 특수문자/HTML 정리
  ② Tokenizer        → 단어로 쪼갬 → [The, cats, are, Running]
  ③ Token filter     → 변형:
       • lowercase    : running
       • stopword 제거 : (the, are 삭제)
       • stemming(어간): cat, run
  최종 색인: [cat, run]
```

**핵심 = 색인 시점과 검색 시점에 "같은 분석기"를 적용** → "running"으로 "run"을 찾는 마법:

```
문서 색인: "I am running" → [run]   (stemming)
검색어:    "runs"         → [run]   (같은 stemming)
→ 둘 다 "run" 토큰 → 매칭!
```

`LIKE`는 글자 그대로라 "running"≠"run"인데, 분석기가 **둘 다 "run"으로 정규화**한다. 동의어(car↔automobile), **한국어 형태소 분석**("고양이들이"→"고양이", ES는 `nori` 분석기)도 여기서.

### 관련도 — BM25

ES는 매칭 여부만이 아니라 **"검색어와 얼마나 관련 있나"** 를 점수로 매겨 정렬한다. BM25 직관:

- **TF**(이 문서에 검색어가 자주 나오나) ↑ → 점수↑
- **IDF**(그 단어가 전체에서 희귀한가) ↑ → 점수↑ ("the"는 흔해 가치 낮고, "Redis"는 희귀해 가치 높음)
- 문서 길이 보정 (짧은 문서에서 매칭된 게 더 관련)

→ 구글처럼 "가장 관련 있는 것 위로". `LIKE`엔 아예 없는 개념.

### Vector search (시맨틱)

ES도 임베딩 벡터를 저장해 **kNN(최근접 이웃) 시맨틱 검색**을 지원한다(= 오늘 [pgvector](PostgreSQL-TimescaleDB.md)와 같은 개념). 요즘은 **키워드 매칭(BM25) + 의미 유사도(벡터)를 결합**한 **하이브리드 검색**이 트렌드(RAG·시맨틱 검색).

## PG로 ES 흉내 — "Just use Postgres"의 검색 버전

PG 확장으로 검색을 흡수하는 방법들:

| 방법 | 무엇 | 대체 |
|------|------|------|
| 내장 FTS (tsvector+GIN) | 기본 전문검색 | 역색인+기본 분석 |
| pg_trgm | trigram 유사검색(오타) | 퍼지 |
| **ParadeDB (`pg_search`)** | **BM25를 PG 안에** (Tantivy 기반) | ES의 BM25 |
| **ZomboDB** | PG 인덱스 뒤에 **실제 ES 백엔드** | ES를 PG 인터페이스로 |
| pgvector | 벡터 시맨틱 | ES vector search |

**여전히 ES 우위:** 대규모 분산, near-realtime 색인, 풍부한 검색 UX(패싯·자동완성·하이라이팅·다국어 형태소). ParadeDB는 신생이라 ES만큼 성숙하진 않음.

→ **"검색이 부가 기능 + 중소 규모 + 이미 PG" → PG 확장으로 충분. "검색이 제품의 핵심 + 대규모" → ES.** (오늘 "PG로 시작, 한계 오면 분리"가 검색에도 적용)

## 폴리글랏 퍼시스턴스 — 왜 PG + Mongo + ES 셋을?

PG 확장으로 검색·벡터·시계열을 다 흡수할 수 있는데도 데이터/용도마다 특화 DB를 쓰는 패턴이 **Polyglot Persistence**다.

```
PostgreSQL    → 정형·관계·트랜잭션 (source of truth, 원본)
MongoDB       → 유연 스키마 비정형 (소스마다 구조 다른 문서)
Elasticsearch → 검색·관련도·분석 (PG/Mongo의 파생 색인)
```

**얻는 것:** 각 워크로드를 특화 엔진이 최적 처리.
**잃는 것 — 동기화 비용:**
- PG/Mongo → ES로 **색인을 계속 흘려보내야** 함 (CDC, 메시지 큐, 배치 / dual write).
- **eventual consistency** — ES는 복제본이라 약간 지연(방금 쓴 게 검색에 바로 안 뜰 수 있음).
- **장애 지점·운영 복잡도 증가** (= [Shopify blast radius](../아티클/Shopify-재고예약-스케일링.md)). DB 3개 = 모니터링·백업 3배.
- 단, **ES는 source of truth가 아님** — 원본은 PG/Mongo. ES가 날아가도 **재색인**하면 됨(장애가 덜 치명적).

### 판단 — "그 DB만의 특화 기능이 진짜 필요한가"

- **ES**는 검색이 핵심이면 정당(PG FTS로 안 되는 대규모·UX).
- **Mongo**는 "**PG의 JSONB로 안 됐나?**"를 의심하라 — 흔한 over-engineering. 데이터가 사실 정형에 가깝거나 유연성을 JSONB로 흡수 가능했다면 Mongo 없이 PG 하나로 충분했을 수 있다.
- "각 DB를 쓰면 멋있어 보여서" 도입하면 동기화·운영 부담만 늘어난다. **DB 개수를 늘리는 건 능력이 아니라 비용**이고, 각 추가는 그 특화가 진짜 필요할 때만 정당하다.

> **오늘 전체를 관통한 메시지:** *단순함(통합) vs 특화(분리)는 트레이드오프, "필요해지기 전엔 분리하지 마라."* — Shopify(2시스템→1), blast radius(분리도 공짜 아님), Just use Postgres(한계 오면 분리), 폴리글랏(특화↔동기화 비용)이 전부 같은 줄기.

## 핵심 질의응답

**Q. 왜 검색은 ES를 따로 쓰나? `LIKE`로는 왜 안 되나?**
A. `LIKE '%x%'`는 인덱스를 못 타 풀스캔이고(앞 고정 `'x%'`만 인덱스), 더 본질적으로 "문자 그대로 매칭"이라 "running"으로 "run"을 못 찾고 관련도 순 정렬도 없다. ES는 역색인+분석기+BM25로 이를 푼다.

**Q. 역색인만이면 PG로도 되는 거 아닌가?**
A. 맞다. PG도 tsvector+GIN으로 전문검색이 된다. ES의 차별점은 역색인 위에 쌓인 분석기(정규화), BM25(관련도), 분산/검색 UX다. 그래서 검색이 부가 기능이면 PG로 충분하고, 제품의 핵심·대규모면 ES다.

**Q. 분석기는 무슨 처리를 하나?**
A. 색인 전에 character filter→tokenizer→token filter 3단계로 가공한다(소문자화, 불용어 제거, 어간 추출, 동의어, 형태소 분석). 색인·검색에 같은 분석기를 적용해 "running"과 "run"이 같은 토큰으로 매칭된다.

**Q. ES를 PG 확장으로 대체할 수 있나?**
A. ParadeDB(pg_search, BM25를 PG 안에)나 ZomboDB(PG 뒤에 실제 ES)로 어느 정도 가능하다. 다만 대규모·near-realtime·검색 UX는 여전히 ES 우위고 ParadeDB는 신생이다.

**Q. PG로 다 되는데 왜 techfeed는 PG+Mongo+ES 셋을 쓰나?**
A. 각 특화(정합성/유연스키마/검색)를 얻는 대신 데이터 동기화 비용(CDC, eventual consistency, 운영 복잡도)을 치른다. ES는 검색이 핵심이면 정당하지만, Mongo는 PG JSONB로 충분했을 수 있어 "굳이?"를 의심할 만하다.

## 주의사항 / 자주 하는 실수

- **ES를 source of truth로 착각** — ES는 파생 색인. 원본은 PG/Mongo, ES는 재색인 가능.
- **색인/검색 분석기 불일치** — 다른 분석기를 쓰면 매칭이 안 된다. 같은 분석기 적용.
- **동기화 지연을 강한 일관성으로 기대** — eventual consistency. 방금 쓴 게 검색에 바로 안 뜰 수 있음.
- **검색 부가 기능에 ES 도입** — 중소 규모면 PG FTS로 충분. DB 추가는 비용.
- **Mongo·ES를 "멋있어서" 도입** — 특화가 진짜 필요한지 먼저 따져라.

## 참고

- [Elasticsearch 공식 문서](https://www.elastic.co/guide/)
- [MongoDB — 문서 DB](MongoDB.md) — 같은 시리즈, 폴리글랏의 문서 계층
- [PostgreSQL & TimescaleDB](PostgreSQL-TimescaleDB.md) — GIN(=역색인), FTS, pgvector, 확장 생태계
- [Redis 캐싱](Redis-캐싱.md) · [Shopify 재고 예약](../아티클/Shopify-재고예약-스케일링.md)
