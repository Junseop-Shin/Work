# Redis — 캐싱 & 인메모리 데이터 스토어

> 2026-06-05 | Redis, 캐싱, 인메모리, 분산락, Pub/Sub

## 한 줄 요약

Redis는 **메모리에 자료구조를 통째로 올려두고 단일 스레드로 직렬 처리**하는 데이터 스토어다. "빠른 key-value"가 아니라, 원자적 연산이 보장되는 자료구조 서버라는 점이 본질이다.

## 핵심 개념

### 왜 Redis를 쓰는가

DB(PostgreSQL 등)는 디스크 기반이라 한 번의 조회에 디스크 I/O + 쿼리 플래닝 + 락 경합이 들어간다. 반복적으로 읽히는데 잘 안 바뀌는 데이터(세션, 인기글, 집계값)를 매번 DB에서 긁으면 낭비다.

Redis는 그 사이에 끼는 **인메모리 캐시 계층**이다. 추가로 단순 캐시를 넘어 자료구조 자체를 제공해서 — 랭킹(Sorted Set), 분산 락, 큐(List/Stream), 카운터(원자적 INCR) 같은 걸 애플리케이션 로직 없이 처리한다.

핵심 설계 포인트: **단일 스레드 이벤트 루프**. 멀티스레드 락이 없으니 모든 명령이 원자적이고, race condition을 신경 쓸 필요가 없다. (네트워크 I/O는 멀티스레드지만 명령 실행은 직렬.) 대신 O(N) 명령(`KEYS *`, 큰 `SMEMBERS`) 하나가 전체를 블로킹한다 — 이게 가장 큰 함정.

### 자료구조 — 무엇을 언제

| 타입 | 내부 | 대표 명령 | 쓰는 곳 |
|------|------|-----------|---------|
| **String** | 단순 바이트 (max 512MB) | `SET/GET`, `INCR`, `SETNX` | 캐시값, 카운터, 분산 락 |
| **Hash** | field-value 맵 | `HSET/HGET`, `HINCRBY` | 객체 캐싱(유저 프로필), 부분 갱신 |
| **List** | 양방향 링크드리스트 | `LPUSH/RPOP`, `BLPOP` | 큐, 최근 항목 N개 |
| **Set** | 해시셋 | `SADD`, `SINTER` | 태그, 중복 제거, 교집합(공통 친구) |
| **Sorted Set** | skiplist + 해시 | `ZADD`, `ZRANGE`, `ZREVRANK` | **랭킹/리더보드**, 시간순 정렬, rate limiting |
| **Stream** | append-only 로그 | `XADD`, `XREADGROUP` | 이벤트 소싱, Consumer Group 큐 |
| **Bitmap / HyperLogLog** | String 위 비트 연산 | `SETBIT`, `PFADD` | 출석 체크, 대규모 유니크 카운트(오차 0.81%) |

**자주 틀리는 선택:**
- 객체를 통째로 JSON String으로 박으면 → 필드 하나 바꿀 때 전체 직렬화/역직렬화. 부분 갱신이 잦으면 **Hash**.
- "최근 본 글 10개" → List에 LPUSH 후 `LTRIM key 0 9`로 잘라낸다.
- 랭킹은 무조건 Sorted Set. score로 정렬된 상태가 유지돼 `ZREVRANGE`가 O(log N).

### 캐시 패턴

#### Cache-Aside (Lazy Loading) — 가장 흔함

```
읽기:  cache.get(key) → hit면 반환
       miss면 → DB 조회 → cache.set(key, val, ttl) → 반환
쓰기:  DB 업데이트 → cache.delete(key)   ← 갱신이 아니라 삭제
```

- 캐시에 없는 데이터만 적재 → 메모리 효율적.
- **쓰기 시 갱신이 아니라 삭제(invalidation)** 하는 게 정석. 갱신하면 동시 쓰기 시 오래된 값이 덮어쓸 수 있다(race). 삭제하면 다음 읽기가 알아서 최신값을 채운다.

#### Write-Through / Write-Behind

- **Write-Through**: 캐시와 DB를 동시에(동기) 갱신. 일관성↑, 쓰기 지연↑.
- **Write-Behind(Write-Back)**: 캐시에 먼저 쓰고 DB는 비동기로 나중에. 쓰기 빠름, but 장애 시 유실 위험.

대부분의 웹 서비스는 **Cache-Aside + 쓰기 시 삭제**면 충분하다. Write-Behind는 쓰기 폭주(IoT 센서 적재 등)에서나 고려.

### TTL & Eviction

두 개는 다른 메커니즘이다 — 자주 헷갈림:

- **TTL(만료)**: 키마다 설정한 시간이 지나면 삭제. Redis는 lazy(접근 시 확인) + active(주기적 샘플링) 혼합으로 만료 처리.
- **Eviction(축출)**: `maxmemory` 한계에 도달했을 때, TTL과 무관하게 메모리 확보를 위해 키를 쫓아냄.

`maxmemory-policy` 정책 (context7 확인):

| 정책 | 대상 | 기준 |
|------|------|------|
| `noeviction` | — | 쓰기 시 에러 반환 (기본값) |
| `allkeys-lru` | 전체 키 | 가장 오래 안 쓰인 것 |
| `allkeys-lfu` | 전체 키 | 가장 적게 쓰인 것 (빈도) |
| `allkeys-random` | 전체 키 | 무작위 |
| `volatile-lru/lfu/random` | **TTL 있는 키만** | 위와 동일 |
| `volatile-ttl` | TTL 있는 키 | 만료 임박한 것부터 |

- **순수 캐시 용도** → `allkeys-lru` 또는 `allkeys-lfu`. LFU는 "한 번 확 몰리고 안 쓰이는" 데이터에 LRU보다 유리.
- **캐시 + 영속 데이터 혼용** → `volatile-*` (TTL 없는 영속 키는 보호).
- `noeviction` + maxmemory 도달 = 쓰기 전면 실패. 캐시인데 이걸로 두면 장애.

### 영속화 (RDB vs AOF)

인메모리지만 디스크 백업 옵션이 있다. **영속화 = Redis가 "자기" 메모리를 "자기" 디스크에 저장하는 것** — 외부 DB 연동이 아니다.

- **RDB**: ⚠️ **R**edis **D**ata**B**ase — **관계형 DB(RDBMS) 아님!** 특정 시점 메모리를 통째로 덤프한 스냅샷 파일(`dump.rdb`, fork 후 저장). 파일 작고 복구 빠름. but 스냅샷 사이 데이터 유실 가능. (이름이 RDBMS와 같아 가장 헷갈리는 부분.)
- **AOF (Append Only File)**: 모든 쓰기 명령을 로그로 기록. 재시작 시 재실행으로 복구. 유실 적음(fsync 정책에 따라), but 파일 크고 복구 느림. `appendfsync everysec`이 현실적 절충(최대 1초 유실).
- 실무: 둘 다 켜고(AOF 우선 복구), 순수 캐시면 영속화 끄거나 RDB만.

#### 영속화 ≠ 다중화 ≠ 캐싱 아키텍처 (3개 다른 층위)

자주 섞이는 세 개념. 모두 독립적으로 켜고 끈다:

| 개념 | 무엇 | 목적 | 축 |
|------|------|------|----|
| **영속화**(persistence) | 메모리 → **자기 디스크** (RDB/AOF) | 재시작 시 복구 | 시간 축 |
| **다중화**(replication) | 한 노드 → **다른 노드** 복사 (Sentinel/Cluster) | 한 대 죽어도 대체(HA) | 공간 축 |
| **캐싱 아키텍처** | Redis(일부·휘발) + DB(전부·원본) | DB 부하↓, 응답↑ | 계층 |

**핵심 오해 교정:** "Redis 영속화가 빨라서 더 안전한 것"이 아니다. **빠르려고 안전을 일부 포기**한 것. 영속화는 성능을 위해 **비동기/주기적**으로 디스크에 쓰므로, 그 사이 장애나면 그만큼 유실된다.

| | 보장 수준 |
|---|---|
| **PostgreSQL 커밋** | 커밋 응답 받으면 **절대 안 날아감** (WAL 동기 fsync, ACID) |
| **Redis RDB** | 마지막 스냅샷 이후 **다 유실** 가능 |
| **Redis AOF everysec** | 최대 **1초치 유실** |

→ Redis 영속화는 "재시작 시 캐시를 데우는(warm-up)" 용도지 "원본 보관"이 아니다. **원본의 source of truth는 항상 DB.**

## 캐시 3대 장애 패턴

면접 단골. 셋 다 "캐시 miss가 DB로 쏟아지는" 문제다.

### 1. Cache Stampede (Thundering Herd)

인기 키의 TTL이 만료되는 순간, 동시에 들어온 수천 요청이 전부 miss → 전부 DB로 직행 → DB 폭사.

**대응:**
- **PER(Probabilistic Early Recomputation)**: 만료 직전에 확률적으로 미리 갱신.
- **Lock/Mutex**: miss 시 첫 요청만 락 잡고 DB 조회, 나머지는 대기 후 캐시값 사용.
- **TTL 지터**: 만료 시간에 랜덤 오프셋(±10%)을 줘서 동시 만료를 흩뜨림.

### 2. Cache Penetration (캐시 관통)

**존재하지 않는 키**를 계속 조회 → 캐시에 없으니 매번 DB로 → DB가 매번 "없음" 확인. 악의적 공격(랜덤 ID 폭격)으로도 쓰임.

**대응:**
- "없음(null)"도 짧은 TTL로 캐싱.
- **Bloom Filter**로 "확실히 없는 키"를 DB 가기 전에 차단.

### 3. Cache Avalanche (캐시 사태)

다수의 키가 **동시에 만료**되거나 Redis 자체가 다운 → 트래픽 전체가 DB로.

**대응:** TTL 분산(지터), Redis 다중화(Sentinel/Cluster), 다단계 캐시(로컬 + Redis).

## Pub/Sub vs Stream

둘 다 메시징이지만 보장 수준이 다르다 — **이걸 혼동하면 메시지 유실 사고.**

| | Pub/Sub | Stream |
|---|---------|--------|
| 모델 | fire-and-forget 방송 | append-only 로그 |
| 구독자 없을 때 | **메시지 소멸** | 로그에 남음 |
| 재처리 | 불가 | offset으로 재읽기 가능 |
| Consumer Group | ❌ | ✅ (`XREADGROUP`, ACK) |
| 용도 | 실시간 알림, 캐시 무효화 신호 | 이벤트 큐, 작업 분배 |

- **Pub/Sub은 메시지를 저장하지 않는다.** 구독자가 그 순간 연결돼 있지 않으면 그냥 사라진다. "알림 한 번 놓쳐도 되는" 용도(라이브 채팅 브로드캐스트, 캐시 무효화 전파)에만.
- 유실되면 안 되는 작업 큐는 **Stream**(또는 별도 MQ). Stream은 Consumer Group + ACK로 "처리 완료 확인"과 재분배(`XCLAIM`)를 지원해 사실상 경량 메시지 큐.

### 왜 Redis Pub/Sub을 쓰나 (전용 MQ가 더 강력한데?)

Pub/Sub은 전용 MQ(Kafka/RabbitMQ)와 **경쟁하는 게 아니라 그 아래 틈새**를 채운다.

- **"비싸지 않나?"** → Pub/Sub 메시지는 **저장을 안 해서**(fire-and-forget) 메모리를 거의 안 먹는다. "인메모리라 비싸다"는 캐시 *데이터* 얘기지 Pub/Sub 자체가 아니다.
- 쓰는 이유: ① **이미 Redis가 떠 있어** 추가 인프라 0 (MQ 새로 띄울 운영비용 회피), ② **초저지연**(µs), ③ **압도적 단순**(`SUBSCRIBE`/`PUBLISH`), ④ 놓쳐도 되는 신호엔 신뢰성이 사치.
- 대표 용처: **멀티 서버 WebSocket 팬아웃**(Socket.io Redis adapter), 캐시 무효화 전파, 실시간 대시보드 푸시.

> 한 줄: 강력해서가 아니라 **싸고 가깝고 충분해서** 쓴다.

### Stream vs Kafka — 언제 무엇

Kafka의 라이벌은 Pub/Sub이 아니라 Redis **Stream**이다(저장+ACK). "빠른 비동기 응답"은 Stream·BullMQ도 주므로 Kafka 선택 이유가 아니다. 진짜 갈림길:

| | Redis Stream | Kafka |
|---|---|---|
| 저장 | 메모리(RAM 비쌈, 트림 필요) | **디스크**(대용량 싸게 장기 보관) |
| 처리량 | 중소 규모 | **초당 수백만**, 파티션 수평 확장 |
| 재처리(replay) | 제한적 | **과거 임의 시점부터 재생** |
| 다중 소비자 | 약함 | 같은 로그를 분석·ML·감사가 독립 소비 |
| 운영 | 가벼움(이미 Redis면 0) | 무거움 |

**선택 기준:**
- 놓쳐도 되는 실시간 방송 + 이미 Redis → **Pub/Sub**
- 중소 규모 유실불가 큐 + 이미 Redis → **Stream** (또는 BullMQ)
- 대용량·장기보관·재처리·다중 다운스트림 → **Kafka**
- 복잡 라우팅·보장 전달·워크큐 → **RabbitMQ**

> Kafka를 쓰는 이유는 *속도*가 아니라 **"내구성 있는 대용량 로그 + 재처리 + 다중 소비자"** 다. 그게 필요 없는 규모면 이미 떠 있는 Redis(Stream)로 충분하다.

## 분산 락 — 왜 위험한가

### 기본 패턴 (context7 확인)

```redis
SET lock:resource <unique-token> NX EX 30
```

- `NX`: 키가 없을 때만 성공 → 락 획득은 한 명만.
- `EX 30`: 30초 후 자동 만료 → 락 잡은 프로세스가 죽어도 영영 잠기지 않음(deadlock 방지).
- `<unique-token>`: 해제 시 "내가 잡은 락인지" 확인용. **해제는 반드시 토큰 비교 후 삭제** — 그냥 `DEL`하면 남의 락을 풀 수 있다. 비교+삭제는 원자적이어야 하므로 Lua 스크립트로:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else return 0 end
```

### 왜 위험한가 — 두 가지 근본 문제

1. **만료 vs 작업 시간 경쟁**: 락 TTL이 30초인데 작업이 GC 멈춤/네트워크 지연으로 35초 걸리면 → 락이 먼저 풀리고 다른 프로세스가 락 획득 → **둘이 동시에 임계구역**. 토큰 비교로 "남의 락 해제"는 막아도, 동시 실행 자체는 못 막는다.
2. **단일 노드의 한계**: 마스터에 락 쓰고 복제 전에 마스터가 죽으면, 승격된 복제본엔 락이 없어 다른 클라이언트가 또 획득.

### Redlock과 그 논쟁

Redis 공식은 다중 마스터에 과반수 획득을 요구하는 **Redlock** 알고리즘을 권장(context7 확인).

**작동 방식** — 서로 복제하지 않는 독립 Redis 마스터 N개(보통 5개):
1. 현재 시각(ms) 기록.
2. N개 전부에 같은 `key`+`token`으로 `SET NX PX` 순차 시도. 각 시도엔 **짧은 타임아웃**(락 TTL보다 훨씬 작게)을 걸어 죽은 노드에 안 매달림.
3. **과반수(5개 중 3개)** 성공 **그리고** 전체 소요시간 < TTL → 획득 성공. 유효시간 = `TTL − 소요시간`.
4. 실패하면 모든 노드에 해제 후 포기.

→ 과반수만 살아있으면 동작해 **단일 장애점**을 없앤다.

**논쟁:** Martin Kleppmann이 "타이밍 가정(GC 멈춤·시계 드리프트)에 의존해 안전하지 않고, 결국 fencing token이 있어야 하며 그게 있으면 Redlock이 불필요하다"고 반박. Redis 저자(antirez)가 재반박하며 유명한 논쟁이 됐다. **대부분의 서비스는 단일 Redis + SET NX EX + Lua 해제로 충분하고 Redlock까지 가는 경우는 드물다.**

**실무 결론:** 락이 깨졌을 때 **정합성이 진짜 중요한** 작업(결제, 재고 차감)은 Redis 락에만 의존하지 말고 **fencing token**(단조 증가 번호)을 DB 단에서 검증하거나, DB 트랜잭션/유니크 제약으로 최종 방어선을 둬라. Redis 락은 "대부분의 경우 중복 실행을 줄이는" 최적화로 보는 게 안전하다.

## Redis vs Memcached

| | Redis | Memcached |
|---|-------|-----------|
| 자료구조 | 다양(List/Set/ZSet/Stream...) | String만 |
| 영속화 | RDB/AOF | 없음(순수 캐시) |
| 스레드 | 단일(코어 1개 위주) | 멀티스레드 |
| 메모리 효율 | 약간 무거움 | 단순 캐시엔 더 효율적 |
| 기능 | Pub/Sub, 트랜잭션, Lua, 클러스터 | 거의 없음 |

- **단순한 string 캐시 + 멀티코어로 처리량 극대화**만 필요 → Memcached가 더 가볍고 빠를 수 있다.
- 그 외 거의 모든 경우 → Redis. 랭킹/락/큐/Pub-Sub 하나라도 필요하면 선택지가 없다. 현업은 사실상 Redis 기본.

## devops-monitor / IoT 적용 패턴

- **최신값 캐싱**: 센서 현재값(`HSET device:{id} temp 23.5`)을 Redis Hash에 두고 대시보드는 Redis만 읽음 → DB 부하 분리.
- **Rate limiting**: `INCR` + `EXPIRE`로 "1분에 N요청" 제한. 또는 Sorted Set으로 sliding window.
- **실시간 알림 팬아웃**: 임계치 초과 이벤트를 Pub/Sub으로 대시보드 WebSocket에 방송(놓쳐도 되는 라이브 신호).
- **작업 큐**: 무거운 집계/알림 발송은 Stream(또는 BullMQ — Redis 기반)으로 비동기 처리.
- 단, **원천 시계열 데이터는 TimescaleDB가 source of truth**, Redis는 핫 캐시 계층. → [PostgreSQL & TimescaleDB](PostgreSQL-TimescaleDB.md)

## 핵심 질의응답

**Q. 유저 프로필 캐싱은 String(JSON) vs Hash 중 뭐가 나은가?**
A. 부분 갱신이 잦으면 Hash(필드 하나만 `HSET`, 직렬화 비용↓). 반대로 항상 객체를 통으로 읽고 부분 갱신이 거의 없으며 다른 서비스가 같은 JSON을 공유하면 String이 더 단순하다. 단 Hash의 TTL은 키 단위로만 걸린다(필드별 만료는 7.4+ 특수 케이스).

**Q. `INCR`이 race condition을 막는 원리는?**
A. 애플리케이션에서 GET→+1→SET을 하면 두 클라가 같은 값을 읽고 덮어써 lost update가 난다. `INCR`은 읽기+증가+쓰기를 Redis 서버 안에서 명령 하나로 처리하고, 싱글스레드라 명령들이 직렬화되므로 원자적이다. 여러 명령을 묶어 원자화하려면 MULTI/EXEC·WATCH·Lua.

**Q. Cache-Aside에서 쓰기 시 캐시를 갱신하지 않고 삭제하는 이유는?**
A. 동시 쓰기 상황에서 "새 값으로 갱신"하면 늦게 도착한 오래된 값이 최신값을 덮어쓸 수 있다(stale). 삭제하면 다음 읽기가 DB에서 최신값을 다시 적재하므로 안전하고, 수정 후 아무도 안 읽으면 헛수고도 없다.

**Q. 인기 키 TTL이 만료되는 순간 동시 요청이 몰리면?**
A. 전부 miss나 DB로 직행하는 Cache Stampede(Thundering Herd). TTL 지터(랜덤 오프셋) + miss 시 첫 요청만 락 잡고 DB 조회로 대응. (없는 키 반복 조회 = Penetration, 대량 동시 만료·Redis 다운 = Avalanche.)

**Q. "최근 검색어 10개를 중복 없이 최신순"엔 어떤 자료구조?**
A. Sorted Set, score를 timestamp로. 같은 검색어는 멤버가 유니크라 `ZADD`가 score만 갱신(중복 제거) + 최신순 정렬이 동시에 된다. List는 중복 제거 불가, Set은 순서 불가.

**Q. Pub/Sub으로 작업 큐를 만들면 안 되는 이유는?**
A. Pub/Sub은 fire-and-forget이라 메시지를 저장하지 않는다. 구독자가 그 순간 연결돼 있지 않으면(재접속 등) 그 사이 메시지는 소멸한다. 유실 불가 큐는 Stream(Consumer Group + ACK) 또는 전용 MQ.

**Q. 전용 MQ가 더 강력한데 왜 Redis Pub/Sub을 쓰나?**
A. 이미 Redis가 떠 있어 추가 인프라가 0이고, 초저지연·단순하며, 놓쳐도 되는 신호엔 신뢰성이 사치이기 때문. fire-and-forget이라 메모리도 거의 안 먹는다. 멀티 서버 WebSocket 팬아웃이 대표 용처.

**Q. 분산 락에 TTL을 거는데도 왜 안전하지 않나?**
A. TTL은 deadlock(락 잡은 프로세스 사망 시 영구 잠김)을 막을 뿐이다. 작업이 TTL보다 오래 걸리면(GC 멈춤 등) 락이 먼저 풀려 두 프로세스가 동시에 임계구역에 들어갈 수 있다. 정합성이 중요하면 fencing token이나 DB 제약으로 최종 방어해야 한다. ([Shopify 사례](../아티클/Shopify-재고예약-스케일링.md)가 실제로 이 결론으로 Redis→MySQL 이전.)

**Q. Redis 영속화는 외부 DB랑 연동하는 건가?**
A. 아니다. 영속화는 Redis가 자기 메모리를 자기 디스크에 저장하는 것(RDB 스냅샷/AOF 로그)이고, 외부 DB와 함께 쓰는 건 별개의 캐싱 아키텍처다. 또 영속화(시간 축 복구)와 다중화(replication, 공간 축 HA)도 다른 개념이다.

**Q. Redis 영속화가 빠르니까 DB보다 안전한가?**
A. 반대다. 빠르려고 안전을 일부 포기한 것. 성능을 위해 비동기/주기적으로 디스크에 쓰므로 그 사이 장애나면 유실된다(RDB는 스냅샷 단위, AOF everysec은 최대 1초). DB 커밋은 동기 fsync로 보장된다. 원본은 항상 DB.

## 주의사항 / 자주 하는 실수

- **`KEYS *` 운영 DB에서 금지** — 단일 스레드를 O(N) 동안 블로킹. 키 스캔은 `SCAN`(커서 기반)으로.
- **모든 키에 TTL을 안 걸면** 메모리가 무한정 차다가 eviction/OOM. 캐시 키엔 반드시 TTL.
- **큰 객체를 String JSON으로** 박고 부분 갱신 반복 → Hash로 바꿔야 직렬화 비용이 준다.
- **Pub/Sub을 신뢰성 큐로 착각** — 가장 흔한 사고.
- **분산 락 해제 시 무조건 DEL** — 남의 락을 푼다. 토큰 비교(Lua)로 해제.
- `maxmemory-policy`를 `noeviction`(기본)으로 둔 캐시 → 메모리 차면 쓰기 전면 실패.

## 참고

- [Redis 공식 문서](https://redis.io/docs/latest/)
- [Eviction policies](https://redis.io/docs/latest/develop/reference/eviction/)
- [Distributed Locks with Redis (Redlock)](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/)
- [PostgreSQL & TimescaleDB](PostgreSQL-TimescaleDB.md) — 동일 시리즈(DB), 원천 저장소 계층
- [인증 종합 (세션/JWT/OAuth)](../인증/인증-JWT-OAuth-OIDC.md) — Redis 세션 스토어 연결
