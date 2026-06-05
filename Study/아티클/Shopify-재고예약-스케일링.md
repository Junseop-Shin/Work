# Shopify — 재고 예약 시스템 스케일링 (Redis → MySQL)

> 2026-06-05 | Shopify, 재고예약, SKIP LOCKED, 격리수준, 핫로우, 동시성
>
> 원문: [Scaling Inventory Reservations](https://shopify.engineering/scaling-inventory-reservations) (Shopify Engineering)

## 한 줄 요약

Shopify가 재고 예약을 **Redis(예약) + MySQL(원장) 2-시스템에서 MySQL 단일 ACID 시스템으로** 옮긴 사례. 핵심 교훈은 *"Redis vs RDB"가 아니라 핫 로우 경합을 어떻게 분산하느냐*였고, 그 답이 **unit당 row + `SKIP LOCKED` + bounded pool**이었다.

> 이 문서는 [Redis 캐싱](../DB/Redis-캐싱.md)과 [PostgreSQL & TimescaleDB](../DB/PostgreSQL-TimescaleDB.md)에서 배운 개념(분산 락의 한계, 격리 수준, 핫 로우, 캐시 stampede)의 종합 실전판이다.

## 배경 — 무엇이 문제였나

재고 예약은 결제 진행 중 "이 상품을 잠깐 잡아두는(short hold)" 동작이다. 오버셀링(두 명이 같은 재고를 사감)을 막아야 한다. 두 단계로 동작:

- **Reserve(예약)**: 결제 시작 시 재고를 잠시 unavailable로.
- **Claim(청구)**: 결제 성공 시 원장에서 영구 차감.

**기존 구조의 결함:**

```
Redis  → 예약 수량 보유 (DECR/INCR)
MySQL  → 진짜 재고 원장(ledger)
```

두 시스템이 **분리돼 있어 reserve와 claim을 원자적으로 묶을 수 없었다.** Redis `DECR`과 MySQL UPDATE가 별개 트랜잭션이라:
- 결제는 성공했는데 재고가 안 잡히거나(언더셀링),
- 재고는 깠는데 예약 정리가 안 되는

실패 모드가 존재했다. → **2개 데이터스토어에 걸친 ACID가 불가능**한 게 근본 문제.

**스케일 감각:** Black Friday 2025 피크 분당 **$5.1M** 매출, 美 이커머스의 14%+.

## 왜 애초에 Redis로 예약을 관리했나 (핵심 통찰)

흔한 오해: "Redis가 빠른 캐시라서." → **틀림.** 진짜 이유는 **조회 속도가 아니라 동시성**이다.

재고 차감의 본질:

> **같은 상품 하나를 수천 명이 동시에 깎으려 든다 = hot row(핫 로우)**

전통 RDB에서 `UPDATE inventory SET qty = qty - 1 WHERE item_id = 42`를 플래시 세일에 날리면, 수천 트랜잭션이 **그 한 row의 락**을 잡으려 줄을 선다. row-level lock 직렬화로 처리량이 폭락. 이게 RDB의 고질적 약점이다.

Redis는 **싱글스레드 + 원자적 `DECR`** 이라 락 없이 명령이 직렬 처리된다. 그래서 핫 로우 경합을 우아하게 피했다. → **Redis를 쓴 진짜 이유 = "락 경합 없는 원자적 카운터".** ([Redis 캐싱 — 싱글스레드/원자성](../DB/Redis-캐싱.md) 참고)

## 해결 — MySQL 단일 시스템 재설계

"RDB로 되돌리면 다시 핫 로우로 느려질 텐데?"라는 의문의 답: **순진하게 되돌린 게 아니라 데이터 모델을 새로 설계했다.**

### ① 수량 컬럼을 버리고 "unit당 row 1개"

```sql
-- ❌ 순진한 방법: 모두가 같은 row 한 개에 줄서기 (핫 로우 재발)
UPDATE inventory SET qty = qty - 1 WHERE item_id = 42;

-- ✅ Shopify: 낱개 행이 여러 개, 안 잠긴 아무 자리나 집기
SELECT id FROM inventory_units
WHERE inventory_item_id = 42 AND reserved = false
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

- **1 row = 1 낱개 (수량 컬럼 없음).** 행을 통째로 잠그는 것 = 정확히 1개 예약.
- 만약 한 행에 수량 5를 넣으면(`qty=5`), 그 행을 5명이 동시에 깎으려 또 락 경합 → 핫 로우 부활. 그래서 **반드시 1행=1개.**

### ② `SKIP LOCKED` — 핫 로우 경합 회피

```
재고 행:  [1✓잠김] [2✓잠김] [3 빈] [4 빈] [5✓잠김] [6 빈]

구매자A: LIMIT 1 FOR UPDATE SKIP LOCKED → 1,2 건너뛰고 3번 집음 (대기 0)
구매자B: 동시 실행                      → 1,2,3 건너뛰고 4번 집음 (대기 0)
```

`SKIP LOCKED`는 "잠긴 행은 **기다리지 말고 건너뛰어**" → 구매자들이 서로 다른 빈 행을 즉시 집어 줄을 안 선다. Redis의 "락 없는 동시성"을 RDB에서 **경합 대상을 여러 행으로 쪼개** 재현한 것.

| 락 충돌 옵션 | 잠긴 행을 만나면 |
|------|------|
| `FOR UPDATE` | 기다림 (직렬화) |
| `FOR UPDATE NOWAIT` | 에러 |
| `FOR UPDATE SKIP LOCKED` | 건너뛰고 다음 행 |

> ⚠️ SKIP LOCKED는 "모든 행을 정확히 본다"를 포기(건너뛰니까). 정확한 `COUNT`엔 부적합, "빈 자리 하나면 됨"인 큐/풀에만.

### ③ Bounded pool — item/location당 최대 1000행

"1 row = 1 unit" 모델은 재고가 100만이면 row도 100만이 될 수 있다. 그래서 **풀을 1000행으로 캡**:

```
원장(ledger): inventory = 5000      ← 진짜 재고는 "숫자"로 보관
      │ 1000개만 낱개 행으로 펼침
      ▼
예약 풀 (최대 1000행): [unit][unit]...[unit]   ← 각 행 = 1개
      ↑ 소진되면 원장에서 보충(replenish)
```

- 5000개 재고면 → **1000개만** 낱개 행으로 펼치고, 나머지 4000개는 원장에 숫자로 대기.
- 풀 소진 시 원장에서 다시 1000개 보충.
- **왜 1000?** "경합 분산(행 많을수록↑) ↔ 테이블 크기(행 많을수록 비대)"의 균형점. 동시 구매자가 1000 넘는 경우는 드물어 SKIP LOCKED가 거의 항상 빈 자리를 찾는다.
- 1000은 **최대 캡**이지 항상 채우는 게 아님. 한산한 상품은 행이 거의 없고, 핫한 상품만 1000까지 참.

### ④ 테이블은 하나 — 복합 PK로 상품 구분

상품마다 테이블을 만들지 않는다(안티패턴). 한 테이블에 모든 상품/위치의 unit 행이 섞이고 **복합 기본키**로 구분:

```sql
PRIMARY KEY (shop_id, inventory_item_id, inventory_group_id, id)
```

- 필터 컬럼이 PK의 일부라 특정 상품 행을 인덱스로 즉시 조회 + **락 수가 행당 2개→1개로 감소.**
- 행 수 ≈ `상품수 × 위치수 × (최대 1000)`. 수만~수십만 행은 RDB가 인덱스로 가볍게 처리.

### ⑤ 격리수준 변경 — `REPEATABLE READ` → `READ COMMITTED`

MySQL InnoDB 기본 `REPEATABLE READ`는 **gap lock**(인덱스 행 사이 "틈"까지 잠가 팬텀 리드 방지, `supremum lock`은 마지막 행 이후 무한대 갭)을 건다.

**문제:** 풀이 비어 보충 INSERT를 하려는데, 동시에 `SELECT ... FOR UPDATE`가 범위를 잡고 있으면 → gap lock이 보충 INSERT를 막아 블로킹/데드락.

**해결:** `READ COMMITTED`는 gap lock을 거의 안 건다(매칭 행만 잠금) → 보충 INSERT와 예약 SELECT가 동시 진행. SKIP LOCKED가 "빈 행 하나"만 집으므로 팬텀 리드는 애초에 문제 안 됨. (기본값이 아니라 프레임워크의 트랜잭션별 격리수준 지원 필요.)

> PostgreSQL은 **기본이 이미 READ COMMITTED**라 이 단계가 불필요할 수 있다. PG의 RR은 gap lock이 아닌 스냅샷 방식이라 메커니즘도 다름. → MySQL 특유의 문제. ([격리 수준](../DB/PostgreSQL-TimescaleDB.md) 참고)

## 반전 — 진짜 병목은 "엔진이 아니라 배관"

쿼리·락을 다 최적화했는데도 플래시 세일에서 **큐잉(지연)** 발생. 그런데 **CPU는 한가**했다 (writer <50%, reader <16%). "CPU 여유 있는데 왜 느려?"

**범인:** 예약 로직이 아니라 **같은 DB 커넥션 풀을 공유하는 다른 체크아웃 코드**(cart·payment·order)가 커넥션을 너무 오래 쥐고 있었다. 풀이 고갈되니 예약 쿼리가 **커넥션을 못 얻어** 줄을 섰다. 쿼리는 빠른데 커넥션을 기다리느라 느렸던 것.

**발견 방법 (관찰 가능성):**
- SQL에 주석 태그: `/* conn_tag:checkout_completion */`
- ProxySQL 레이어에서 "어떤 호출자가 커넥션을 얼마나 오래 쥐는지" 추적 → 진짜 범인 attribution.

**핵심 교훈:** *"숫자가 안 맞으면(낮은 CPU + 높은 큐잉) 전체 경로를 계측하라. 답은 엔진이 아니라 배관에 있다."* 예약은 cart·payment·order와 DB를 공유하므로, **고립시켜 최적화하면 더 큰 자원 경합을 놓친다.**

## 심화 — 자주 하는 오해 3가지

### ① "RDB가 느린 건 디스크 I/O 때문" → 아니다, 락 경합 때문

핫 로우 UPDATE가 느렸던 건 디스크가 느려서가 아니라 **수천 트랜잭션이 같은 row의 락을 두고 직렬화**돼서다. 게다가 자주 읽는 핫 데이터는 **버퍼 풀(메모리 캐시)** 에 떠 있어 디스크까지 안 간다(PG `shared_buffers`/InnoDB buffer pool). Shopify에서 **CPU가 한가했다**는 게 I/O·연산이 병목이 아니었다는 증거. → I/O는 버퍼 풀이 흡수하고, 진짜 적은 락(초기)→커넥션 풀(최종)이었다. ([버퍼 풀 상세](../DB/PostgreSQL-TimescaleDB.md))

### ② "멀티스레드로 극복했다" → 멀티스레드는 원래 있었고, SKIP LOCKED가 풀어준 것

MySQL·PostgreSQL은 **원래 멀티스레드**라 여러 트랜잭션을 병렬 처리한다 — 능력이 새로 생긴 게 아니다. 문제는 **핫 로우에선 그 병렬성이 무의미**했다는 것:

> **계산대 비유** — 멀티스레드 = 계산대 10개. 핫 로우 = 손님이 전부 한 계산대에 줄 섬 → 1개만 일하고 9개는 논다. **SKIP LOCKED + unit-row** = 손님을 여러 계산대로 분산 → 비로소 10개가 다 일한다.

즉 SKIP LOCKED는 멀티스레드를 *추가*한 게 아니라, **잠겨있던 멀티스레드 병렬성을 가로막던 락 경합을 제거**했다.

### ③ "결국 Redis가 더 빠르니 RDB는 열등" → 속도 우위가 이 문제에선 결정적이지 않았다

raw 단일 연산 latency는 Redis(µs)가 RDB(ms)보다 빠른 게 맞다. 하지만:
- **"더 빠르다" ≠ "충분히 빠르다"** — 사용자 체감 결제 응답은 수백 ms~초. 0.001ms vs 1ms는 사람이 못 느낀다. 충분히 빠른 영역에선 그 위의 가치(정합성)가 중요.
- **Redis의 속도는 정합성을 포기한 대가** — 2-시스템 비원자성이 그 속도의 일부였고, 그게 버그를 낳았다. Shopify는 "약간 느려도 오버셀링 0"을 택했다. 속도를 양보하고 정확성을 산 것(트레이드오프지 패배가 아님).
- **진짜 병목은 DB 속도가 아니라 커넥션 풀** — Redis로 뒀어도 안 풀렸을 문제.

> **스포츠카 vs 트럭:** 스포츠카(Redis)가 빠른 건 맞지만, 정합성 보장된 재고를 안전하게 나르는 덴 트럭(RDB)이 맞고, 도심 속도제한(체감 응답시간) 안에선 둘 다 충분히 빠르다. 선택 기준은 최고 속도가 아니라 *무엇을 안전하게 싣느냐*.

## 핵심 질의응답

**Q. 왜 처음엔 예약을 Redis로 관리했나? 빠른 조회 때문?**
A. 조회 속도가 아니라 동시성 때문. 재고 차감은 같은 상품 row를 수천 명이 동시에 깎는 핫 로우 작업이라 RDB의 row-lock 경합이 치명적이다. Redis의 싱글스레드 원자 `DECR`이 락 없이 이를 처리해줬다.

**Q. RDB로 되돌리면 Redis만큼 빠르지 않을 텐데?**
A. 순진하게 되돌리면 맞다. 하지만 Shopify는 "수량 컬럼 1개"를 "unit당 row + SKIP LOCKED + bounded pool"로 재설계해 경합을 분산했다. 모델을 바꾸니 RDB로도 충분한 반응성 + ACID 정합성을 얻었다(writer CPU <50%).

**Q. 5000개 재고를 5씩 1000개 행으로 쪼갠 건가?**
A. 아니다. 수량을 묶으면 그 행이 다시 핫 로우가 된다. 1행=1개(수량 컬럼 없음)로, 5000개 중 1000개만 낱개 행으로 펼치고 나머지는 원장에 숫자로 둔다. 풀이 비면 원장에서 보충한다.

**Q. 상품 100개면 100×1000 행? 테이블을 분리하나?**
A. 테이블은 하나, 복합 PK 인덱스로 상품 구분(테이블 분리는 안티패턴). 1000은 최대 캡이라 한산한 상품은 행이 거의 없다. 설령 수만 행이어도 RDB는 인덱스로 가볍게 처리한다.

**Q. PostgreSQL에도 SKIP LOCKED가 있나?**
A. 있다(9.5+, 2016). 문법도 동일. 오히려 PG는 SKIP LOCKED로 "DB 자체를 경량 작업 큐로" 쓰는 패턴(pg-boss, graphile-worker)이 인기다.

## 주의사항 / 일반화 포인트

- **정합성이 핵심인 작업은 Redis 락/카운터에만 의존하지 마라.** 최종 방어선(ACID 트랜잭션, DB 제약)을 DB에 둬라. ([분산 락 — 왜 위험한가](../DB/Redis-캐싱.md) 참고)
- **"RDB는 느리다"는 모델 탓일 때가 많다** — 핫 로우를 행 분할 + SKIP LOCKED로 분산하면 RDB로도 고동시성 가능.
- **2개 데이터스토어에 걸친 원자성은 어렵다** — 가능하면 단일 트랜잭션 경계로.
- **성능 문제는 직접 경로 밖**(공유 커넥션 풀)에 있을 수 있다 — 전체 경로를 계측하라.

## 참고

- [원문: Scaling Inventory Reservations](https://shopify.engineering/scaling-inventory-reservations)
- [Redis 캐싱](../DB/Redis-캐싱.md) — 싱글스레드/원자성, 분산 락의 한계
- [PostgreSQL & TimescaleDB](../DB/PostgreSQL-TimescaleDB.md) — SKIP LOCKED, 격리 수준, MVCC
