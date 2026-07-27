# PostgreSQL & TimescaleDB — RDB 중 PostgreSQL의 특징

> 2026-06-05 | PostgreSQL, TimescaleDB, MVCC, 인덱스, 격리수준, 시계열, 확장
>
> **스코프 노트:** 이 문서는 **"RDB 중 PostgreSQL의 차별점"** 에 집중한다. JOIN 전략·EXPLAIN·일반 SQL 같은 **RDB 공통 주제는 별도 "RDB 공통" 문서**로 분리 예정(다음 과제).

## 한 줄 요약

PostgreSQL은 **단일 스토리지 엔진 위에 확장(extension)을 수직으로 쌓아 멀티모델을 흡수**하고, **MVCC를 테이블에 직접 적층**(→ VACUUM이 생명선)하는 객체-관계형 DB다. TimescaleDB는 그 확장성의 대표 사례로, **시간 기준 자동 파티셔닝(hypertable)** 을 얹은 시계열 확장이다.

## 1. 왜 PostgreSQL — MySQL 대비 차별점

출발점이 다르다. MySQL은 **"빠르고 단순한 웹 DB"**, PostgreSQL은 **"표준 제대로 + 복잡한 것도 정확히 + 확장 가능하게"** 로 시작했다. 그래서 PG를 **객체-관계형 DB(ORDBMS)** 라 부른다. 차별점 3기둥:

- **① 기능 깊이** — CTE·윈도우 함수·FULL OUTER JOIN·재귀 쿼리, 풍부한 타입(배열·JSONB·range·기하), 다양한 인덱스.
- **② 확장성(extensibility)** — extension으로 코어를 안 건드리고 능력을 얹음(PostGIS·pgvector·TimescaleDB). "하나로 다 되는" DB.
- **③ 엄격함/정합성** — 표준 SQL, **트랜잭션 DDL**(스키마 변경도 롤백), 강한 제약.

### 스토리지 엔진 — 가장 구조적인 차이

PG vs MySQL의 가장 근본적 차이는 **다양성을 얻는 방식**이다:

| | 다양성 확보 방식 | 예 |
|---|---|---|
| **MySQL/MariaDB** | 스토리지 엔진을 **수평 교체** (쿼리층과 스토리지층 분리) | `ENGINE=InnoDB`(트랜잭션)/`MyISAM`/MariaDB의 ColumnStore·MyRocks |
| **PostgreSQL** | 단일 엔진 위에 extension을 **수직 적층** | PostGIS·pgvector·TimescaleDB |

```sql
-- MySQL/MariaDB: 테이블마다 엔진 선택
CREATE TABLE orders (...) ENGINE=InnoDB;   -- 트랜잭션/row lock
CREATE TABLE logs   (...) ENGINE=MyISAM;   -- 옛날, 테이블 lock
```

→ **MySQL = 엔진 교체(모듈식), PostgreSQL = 확장 적층(한 코어에 부품 쌓기).** TimescaleDB가 "PG에 시계열을 extension으로 얹은" 대표 예다.

> **프로젝트에서 PG 역할:** techfeed가 PG+Mongo+ES를 같이 쓴다면 — **PG = 트랜잭션·정합성이 중요한 정형 핵심 데이터(source of truth)**, Mongo = 유연 스키마 비정형, ES = 전문검색. PG를 고른 건 ③ 정합성 때문. (= [Shopify](../아티클/Shopify-재고예약-스케일링.md)의 "원본은 ACID DB에" 원칙)

## 2. MVCC & VACUUM — PG의 동시성 구현

**MVCC(다중 버전 동시성 제어)** — 읽기/쓰기가 서로 안 막게:

- UPDATE는 기존 행을 **제자리에서 안 고친다.** dead로 표시 + **새 버전 튜플을 같은 테이블에 추가**. 각 튜플에 `xmin`/`xmax`(생성·삭제 트랜잭션 ID).
- 각 트랜잭션은 자기 스냅샷에서 보이는 버전만 읽음 → **읽기가 쓰기를 안 막고, 쓰기가 읽기를 안 막는다.**

### PG vs MySQL — dead 버전을 어디 두나 (결정적 차이)

| | dead 버전 위치 | 결과 |
|---|---|---|
| **PostgreSQL** | **메인 테이블 안**에 새 버전 적층 | 테이블이 직접 부풀어(bloat) → **VACUUM이 생명선** |
| **MySQL InnoDB** | **undo log**(별도 영역) | 메인은 in-place 갱신 |

**VACUUM** = dead tuple을 회수해 공간 재사용 가능하게. autovacuum이 자동 처리.
- **공간을 OS에 반납하진 않음**(재사용만). 물리 축소는 `VACUUM FULL`(테이블 락 — 운영 중 위험, `pg_repack`으로 대체).
- **XID wraparound** — 트랜잭션 ID 32비트 한계. autovacuum이 freeze로 막지만, 방치하면 **DB 강제 셧다운**. PG 운영 대표 주의사항.

> 핵심: PG의 MVCC는 **테이블에 새 버전을 직접 적층**하므로 **autovacuum이 안 돌면 bloat·wraparound로 무너진다.** "PG = VACUUM 관리가 중요한 DB"라는 평판의 근원.

## 3. 버퍼 풀 — DB 안의 캐시 (성능 기초)

자주 읽는 데이터는 디스크까지 안 가고 **버퍼 풀(메모리 캐시)** 에서 처리된다 (PG `shared_buffers`, MySQL `InnoDB buffer pool`).

- **버퍼 풀도 결국 캐시 → eviction이 돈다.** 메모리가 차면 LRU 계열로 "안 쓰인 페이지"를 내보냄 (PG는 clock-sweep). = [Redis `maxmemory-policy`](Redis-캐싱.md)와 같은 원리.
- **핫 데이터만 적용된다** — 메모리가 한정돼 전체 DB를 못 올린다. 진짜 성능 지표 = **"워킹셋(자주 건드리는 데이터)이 버퍼 풀에 들어가느냐"**. 들어가면 cache hit 99%+, 넘치면 thrashing으로 디스크 I/O 폭증.
- `EXPLAIN (ANALYZE, BUFFERS)`로 **shared hit(버퍼)** vs **read(디스크)** 를 직접 본다.

### 버퍼 풀 vs Redis — "DB 안의 캐시" vs "DB 앞의 캐시"

| | 버퍼 풀 | Redis |
|---|---|---|
| 위치 | DB 안 | DB 앞 |
| 데이터 줘도 거치는 것 | 쿼리 플래닝·락·MVCC 가시성·커넥션 | **키 하나로 즉시**(엔진 통째 건너뜀) |

→ 버퍼 풀이 메모리에서 줘도 여전히 쿼리 엔진을 거친다. Redis는 그걸 건너뛰어 **DB 부하(CPU·커넥션) 자체를 덜어낸다.** 그래서 버퍼 풀이 있어도 [Cache-Aside](Redis-캐싱.md)로 Redis를 앞에 둔다.

## 4. 인덱스 — 5종과 선택

인덱스 = "정렬된 보조 자료구조로 풀스캔을 피하는" 책의 색인. **모든 컬럼에 다 걸면 안 된다** — 쓰기마다 인덱스도 갱신해 **쓰기가 느려지고**, 버퍼 풀을 잠식하며, 카디널리티 낮은 컬럼(boolean 등)은 옵티마이저가 안 쓴다. **읽기 속도를 쓰기·공간과 맞바꾸는 거래**라 선별적으로.

| 종류 | 구조 / 본질 | 잘 맞는 쿼리 |
|------|------|--------------|
| **B-tree** (기본) | 정렬된 트리 | `=` `<` `>` `BETWEEN` `ORDER BY` — 등호·범위·정렬 다 |
| **Hash** | 해시 | `=` 등호만 (순서 없어 범위 불가) |
| **GIN** | **역색인** | 한 행에 값 여럿: JSONB(`@>`)·배열·전문검색(`@@`) |
| **GiST** | 일반화 검색 트리 | 겹침/근접: 기하(PostGIS)·범위 겹침·KNN |
| **BRIN** | 블록 범위 요약(min/max) | **정렬된 거대 테이블**(시계열) — 초경량 |

**핵심 구분:**
- **B-tree가 기본인 이유** = 등호·범위·정렬을 다 커버. Hash는 등호만이라 특수 케이스.
- **GIN = 역색인** — `값 → 그 값을 가진 행 목록`. "한 행에 값이 여럿"(배열·JSONB·검색)일 때. 빌드·쓰기는 느리고 읽기 빠름. (= Elasticsearch 역색인과 같은 원리)
- **GiST = 겹침/거리** — 기하·범위. 지리 extension(PostGIS) 쓸 때 깊이 보면 됨.
- **BRIN = 블록당 min/max만** 기록 → B-tree보다 수백 배 작음. **물리 정렬 = 값 정렬**일 때만 효과(시계열 ⭕, 랜덤 UUID ❌). B-tree가 "정확히 어느 행"이면 BRIN은 "대략 어느 블록".

**복합 인덱스 좌측 prefix 규칙** — `(user_id, created_at)`은 `user_id` 또는 `(user_id, created_at)` 조건엔 쓰이지만 **`created_at` 단독엔 안 쓰인다**(전화번호부: 성을 모르면 이름만으론 못 찾음). 그래서 **등호로 자주 거르는 컬럼을 앞에, 범위 컬럼을 뒤에** 둔다(앞에서 한 점 찍고 뒤에서 범위 훑기).

## 5. 격리수준 & SKIP LOCKED

**격리수준은 트랜잭션의 속성**이지 테이블의 속성이 아니다(테이블마다 다르게 못 건다). 설정: 트랜잭션별(`BEGIN ISOLATION LEVEL ...`, 가장 흔함)·세션별·전역. 첫 쿼리 실행 전에 정해야 함.

| 수준 | 막는 이상현상 | PG 특이점 |
|------|---------|-----------|
| READ UNCOMMITTED | — | PG에선 **RC처럼**(dirty read 없음) |
| **READ COMMITTED** (PG 기본) | dirty read | 각 쿼리가 그 시점 스냅샷 |
| REPEATABLE READ | + non-repeatable, **팬텀까지** | **스냅샷 방식**(MySQL은 gap lock) |
| SERIALIZABLE | + 직렬화 이상 | **SSI**(아래) |

- **기본이 RC** (MySQL은 RR). PG의 RR은 **스냅샷**이라 MySQL의 **gap lock** 과 메커니즘이 다르다.
- **SERIALIZABLE = SSI(Serializable Snapshot Isolation)** — 전통 DB가 락(2PL)으로 구현하는 걸, PG는 **락 없이 낙관적으로**: 스냅샷으로 돌리다 직렬화 위반 충돌을 감지하면 한 트랜잭션을 **abort**. 앱은 재시도 로직만 갖추면 됨.
- **`SKIP LOCKED`(9.5+)** — `FOR UPDATE`는 잠긴 행을 기다리지만, `SKIP LOCKED`는 건너뛰고 다음 행 → 핫 로우 경합 회피. PG는 이걸로 **DB를 경량 작업 큐**로 쓰는 문화가 강하다(`pg-boss`, `graphile-worker`).

> [Shopify가 MySQL gap lock 때문에 RC로 내린 사례](../아티클/Shopify-재고예약-스케일링.md)를, PG는 기본이 RC + RR도 스냅샷이라 덜 겪는다.

## 6. 확장 생태계 & JSONB — 멀티모델

### Extension — 단일 엔진 위에 능력 적층

| 확장 | 추가 능력 | 대체하는 전용 DB |
|------|----------|-----------------|
| **PostGIS** | 지리·공간(GiST) | 지리 전용 |
| **pgvector** | 벡터 임베딩·유사도(HNSW) | Pinecone/Weaviate |
| **TimescaleDB** | 시계열(hypertable) | InfluxDB |
| **Apache AGE** | **그래프**(Cypher) | Neo4j |
| **pg_trgm / Citus** | 유사텍스트 / 분산·샤딩 | — |

→ "검색·시계열·벡터·그래프를 DB 여러 개로" 대신 **PG 하나로 흡수** = "Just use Postgres" 트렌드. **단, 규모·특화가 커지면 전용 DB가 유리**(대규모 전문검색은 ES) — "PG로 시작, 한계 오면 분리".

**그래프 ≠ 벡터 (자주 혼동):**
- **지식그래프 = 그래프 DB(Neo4j / Apache AGE)** — 노드+엣지, **관계 구조 탐색**("친구의 친구").
- **pgvector = 벡터 DB** — 임베딩 **의미 유사도**("비슷한 글"). RAG·시맨틱 검색.
- AI/GraphRAG에서 함께 쓰여 헷갈리지만 **모델이 별개**. PG로 그래프는 Apache AGE나 재귀 CTE(`WITH RECURSIVE`).

### JSONB — PG 안에서 NoSQL

정형은 컬럼, 비정형은 JSONB 컬럼 — 한 테이블에서 둘 다. **"Mongo의 유연함 + PG의 정합성·조인·트랜잭션".**

| | JSON | **JSONB** |
|---|---|---|
| 저장 | 텍스트 그대로 | **바이너리 파싱** |
| 인덱스 | ❌ | **GIN 가능** |
| 실무 | 거의 안 씀 | 기본 |

```sql
SELECT * FROM events WHERE data @> '{"type":"click"}';  -- 포함, GIN 색인
-- -> JSONB추출, ->> 텍스트추출, @> 포함, ? 키존재
```

> 진짜 문서 중심·대규모 유연 스키마면 Mongo가 낫고, 정형이 메인 + 일부만 유연하면 JSONB로 충분(월요일 Mongo 비교).

## 7. TimescaleDB — 시계열 (PG 확장)

### 왜 — 시계열의 문제

시계열(센서·메트릭·로그)은 **시간순 끝없는 INSERT + 최근 위주 조회**. 일반 테이블에 수십억 행이면 인덱스 비대·INSERT 저하. TimescaleDB는 **시간 자동 파티셔닝 + 열 압축**으로 풀고, **PG 확장이라 SQL·조인·인덱스를 그대로** 쓴다(InfluxDB는 전용 언어).

### Hypertable — 자동 시간 파티셔닝

겉보기 한 테이블, 내부는 시간 범위별 **chunk**:

```sql
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL, sensor_id INT, temperature DOUBLE PRECISION
) WITH (tsdb.hypertable, tsdb.partition_column = 'time');
```

→ 시간 조건 쿼리는 **관련 chunk만 스캔**(chunk exclusion). 최근 chunk만 hot하게 유지돼 INSERT 빠름. BRIN 인덱스가 여기 궁합.

### 압축 (Columnstore) — 90%+, 읽기는 자동 복원

오래된 chunk를 **행→열 저장**으로. 같은 컬럼 값이 모여 압축률 폭증. `segmentby`(묶음 기준)·`orderby`로 설정하고 정책으로 자동화.

**읽을 때 동작 (중요):**
- **자동 복원** — 쿼리하면 투명하게 압축 해제. 수동 작업 불필요.
- **핫=비압축(rowstore), 콜드=압축(columnstore).**
- **"압축=읽기 느림"이 아니다** — 워크로드로 갈린다:
  - **분석/집계/범위 스캔 → 오히려 빠름** (디스크 I/O 1/10 + 필요 컬럼만 + 벡터화). I/O 절감이 복원 CPU보다 큰 이득.
  - **단건 포인트 조회 → 약간 느림** (블록 풀어야 함).
- 그래서 **단건 잦은 핫은 비압축, 분석 위주 콜드는 압축** — 콜드의 주 워크로드(분석)엔 압축이 궁합. (ClickHouse가 컬럼 압축으로 분석에 빠른 것도 같은 원리)

### 연속 집계 (Continuous Aggregate)

자주 보는 집계를 **증분 갱신 MV**로 미리 계산(새 데이터분만 점진 갱신):

```sql
CREATE MATERIALIZED VIEW sensor_hourly WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS hour, sensor_id, AVG(temperature) avg_temp
FROM sensor_data GROUP BY hour, sensor_id;

SELECT add_continuous_aggregate_policy('sensor_hourly',
  start_offset => INTERVAL '3 hours', end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');
```

**연속 집계 vs raw 전체 집계 — 차이가 크다.** 한 달 집계 예(센서 100, 1초 주기):

| | 읽을 행 | 속도 |
|---|---|---|
| **raw 전체 집계** | ≈2.6억 행 디코드+연산 | 수 초~수십 초 |
| **연속 집계 합산** | ≈7.2만 행 | 수 ms |

→ **약 3,600배 적은 데이터.** 압축이 분석에 유리해도 **"읽을 행 자체가 수천 배 적은 것"** 을 못 이긴다(가계부 월합계 vs 영수증 1만장 재합산).

⚠️ **집계 함수별 재집계 가능성:**
- **SUM·COUNT·MAX·MIN** → 재집계 가능(시간별 결과를 또 합치거나 최대 고르기).
- **AVG → 함정!** 평균들을 평균내면 틀림(개수 가중치 무시). 연속 집계는 **SUM·COUNT로 분해 저장** 후 `총SUM/총COUNT`로 계산.
- **중앙값·백분위·DISTINCT COUNT** → 결합적이지 않아 근사 알고리즘(t-digest, HyperLogLog) 필요(TimescaleDB Toolkit).

**보존 정책** — `add_retention_policy`로 "30일 지난 raw 삭제, 집계만 장기 보관" → 공간 절약.

### 시계열 DB 비교

핵심 기능(압축/집계/보존/파티셔닝)은 시계열 DB의 **공통 기본기**. 차이는 ① **연속집계 자동증분** ② **범용성(SQL·조인)** ③ **기존 스택 통합**.

| DB | 연속집계 | 범용성 | 특징 |
|---|---|---|---|
| **TimescaleDB** | **◎ 자동증분** | ◎ SQL·조인 | PG 확장 |
| **InfluxDB** | ○ continuous query | △ 전용언어 | 시계열 전용 |
| **Prometheus** | △ PromQL | △ | 모니터링 특화, 장기보관은 Thanos/Mimir |
| **ClickHouse** | ○ MV | ◎ SQL | 컬럼형 OLAP, 분석 최강 |
| **MongoDB TS** | △ 수동($merge) | △ 문서모델 | 유연 스키마 |

**MongoDB Time Series(5.0+):** 버킷팅·압축·TTL은 있으나 **연속집계 자동증분이 약함**. 처리 구분이 핵심:
- **집계 연산**(Aggregation Pipeline) → mongod **내부**.
- **주기적 자동 실행 스케줄러** → **외부**(앱 cron / Atlas Trigger) — Mongo 코어에 내장 스케줄러가 없어서.
- 반면 **TimescaleDB는 연산+스케줄링 둘 다 DB 내부** background worker → **data locality**(데이터 곁 자동화). PG라 PL/pgSQL·pg_cron까지.

> 단 stored procedure는 양날의 검(성능 ⭕, 버전관리·테스트·디버깅 ❌). "처리를 앱에서"인 Mongo 철학도 나름의 장점.

### 실전 적용 검토 — Mongo+PG+배치 → PG+TimescaleDB 통합

**현재 아키텍처(IoT/devops-monitor):**
```
Azure Table Storage(영구원천, 3개월 hot + 압축 아카이빙)
  → MongoDB(5분 raw) → 외부 배치 집계 → PostgreSQL(시/일 집계)
  앱: raw는 Mongo / 집계는 PG 에서 조회
```

**통합안:** raw → Timescale hypertable, 집계 → continuous aggregate, 앱은 PG 한 곳.

**이점:** 외부 배치 제거(가장 큼) · Mongo 제거 · 조회 단일화 · 조인 가능 · 비용↓.

**짚을 점:**
- **5분 주기면 데이터량 작아** → 동기는 *성능*이 아니라 *관리 통합·집계 자동화*. (그게 정확히 Timescale 편의에 맞음)
- **Azure 레이크는 역할이 다름**(장기 원천·DR) → 유지, 운영 계층만 통합.
- **Mongo 유연 스키마**가 필요했으면 JSONB로 흡수(센서는 대개 정형이라 무방).

**핵심 트레이드오프 — blast radius vs 가용성:**
- 통합하면 raw·집계 독립성을 잃어 "한 곳 죽으면 다 죽는" 우려가 생김 (타당).
- 하지만 **"관리 포인트 수" ≠ "가용성"**. 분리도 공짜가 아니다 — 장애 날 곳이 더 많고(외부 배치 실패도 장애), 전체 가용성은 곱이라 더 낮을 수 있다.
- 단일 관리 단위라도 **HA(zone-redundant Primary+Standby 자동 failover) + PITR 백업**이면 SPOF가 아니다. + **Azure 레이크가 최종 복구 안전망.**
- 결론: **"통합(단순함) + HA(가용성)"** — 두 축을 분리해 확보. 물리 인스턴스 분리는 부하가 실증되면 나중 옵션.

> **Redis = 핫 캐시(실시간 최신값), TimescaleDB = source of truth(이력·분석).** ([Redis IoT 패턴](Redis-캐싱.md)과 짝.)

## 핵심 질의응답

**Q. 모든 컬럼에 인덱스를 다 걸면 안 되는 이유는?**
A. 쓰기마다 그 행의 모든 인덱스를 갱신해 **쓰기가 느려지고**, 인덱스가 버퍼 풀을 잠식하며, 카디널리티 낮은 컬럼은 옵티마이저가 안 쓴다. 인덱스는 읽기 속도를 쓰기·공간과 맞바꾸는 거래다.

**Q. `(user_id, created_at)` 인덱스가 `created_at` 단독 조건에 안 먹는 이유는?**
A. 복합 인덱스 좌측 prefix 규칙. user_id로 먼저 정렬되고 그 안에서 created_at이 정렬되므로, 앞 컬럼을 건너뛴 created_at 단독은 못 탄다(전화번호부에서 성을 모르면 이름만으론 못 찾음).

**Q. UPDATE가 행을 제자리에서 안 고친다는 게 무슨 의미인가?**
A. MVCC. 기존 튜플을 dead로 표시하고 새 버전을 같은 테이블에 추가해, 다른 트랜잭션이 자기 스냅샷의 옛 버전을 계속 읽게 한다. 읽기/쓰기가 안 막히는 대신 dead tuple이 쌓여 VACUUM이 필요하다. PG는 테이블에 직접 적층(MySQL은 undo log)이라 VACUUM이 특히 중요하다.

**Q. 버퍼 풀이 있는데 왜 Redis를 또 쓰나?**
A. 버퍼 풀이 메모리에서 데이터를 줘도 여전히 쿼리 플래닝·락·MVCC 가시성·커넥션을 거친다. Redis는 그 엔진을 통째로 건너뛰어 DB 부하(CPU·커넥션) 자체를 덜어낸다. 버퍼 풀은 "DB 안의 캐시", Redis는 "DB 앞의 캐시".

**Q. 압축하면 읽을 때 느려지나?**
A. 자동 복원되며, 항상 느린 건 아니다. 분석/집계/범위 스캔은 디스크 I/O가 1/10로 줄고 필요 컬럼만 읽어 **오히려 빠르다**. 단건 포인트 조회만 블록을 풀어야 해 약간 느리다. 그래서 단건 잦은 핫은 비압축, 분석 위주 콜드는 압축한다.

**Q. 한 달 집계를 연속 집계 vs 압축 raw 전체 집계, 속도 차이가 큰가?**
A. 크다. 연속 집계가 읽을 행이 수천 배 적어(2.6억 vs 7.2만) 압도적으로 빠르다. 압축은 읽을 바이트만 줄일 뿐 모든 행을 거쳐야 한다. 단 SUM/MAX/MIN은 재집계 가능하지만 AVG는 SUM·COUNT로 분해해야 정확하다.

**Q. 그래프 DB(지식그래프)는 pgvector로 하나?**
A. 아니다. 지식그래프는 노드+엣지의 관계 구조 탐색이라 그래프 DB(Neo4j/Apache AGE)의 영역이고, pgvector는 임베딩 의미 유사도(시맨틱 검색)용이다. AI 맥락에서 함께 쓰여 헷갈리지만 모델이 다르다.

**Q. Mongo+PG+배치를 PG+Timescale로 합치면 장애 시 통째로 죽지 않나?**
A. 타당한 우려(blast radius)지만, "관리 포인트 수"와 "가용성"은 다른 축이다. 분리도 장애 지점이 더 많다(배치 실패 등). HA(Primary+Standby 자동 failover)+PITR이면 단일 관리 단위라도 SPOF가 아니고, 영구 원천(데이터레이크)이 최종 복구 안전망이 된다.

## 주의사항 / 자주 하는 실수

- **autovacuum 방치** — bloat 누적, 최악엔 XID wraparound로 강제 셧다운.
- **인덱스 남발 / 복합 인덱스 컬럼 순서** — 쓰기 저하, 좌측 prefix 규칙 무시.
- **`VACUUM FULL`을 운영 중** — 테이블 전체 락. `pg_repack`으로 대체.
- **TimescaleDB에서 시간 조건 없이 쿼리** — chunk exclusion이 안 먹어 전체 chunk 스캔.
- **연속 집계의 AVG를 단순 평균으로 재집계** — 개수 가중치를 무시해 틀린 값.
- **압축 chunk에 단건 포인트 조회를 자주** — 블록 복원 비용. 단건 잦은 데이터는 핫(비압축) 영역에.

## 참고

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/) · [TimescaleDB 문서](https://docs.timescale.com/)
- [Redis — 캐싱 & 인메모리](Redis-캐싱.md) — 핫 캐시 계층, 버퍼 풀과 대비
- [Shopify 재고 예약 스케일링](../아티클/Shopify-재고예약-스케일링.md) — SKIP LOCKED·격리수준·핫로우 실전
- [ORM 패턴 비교 (AR vs DM)](../ORM-ODM/ORM-패턴-비교.md) — 이 DB를 애플리케이션에서 다루는 계층
- *(예정)* RDB 공통 — JOIN 전략 / EXPLAIN / 실행계획
