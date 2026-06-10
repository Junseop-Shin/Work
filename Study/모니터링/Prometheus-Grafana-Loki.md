# Prometheus / Grafana / Loki — 실전

> 2026-06-10 | Prometheus, Grafana, Loki, PromQL, LogQL

## 한 줄 요약

Prometheus는 **타깃의 `/metrics`를 주기적으로 긁어오는(pull) 시계열 DB**, Grafana는 **여러 데이터소스를 쿼리해 한 화면에 그리는 시각화 레이어**, Loki는 **라벨만 인덱싱하는 로그 전용 백엔드**다. 셋은 공통 라벨 모델로 묶인다.

> 개념 토대(3축, 메트릭 타입, RED/USE, 알림 철학)는 → [관찰가능성 개념](관찰가능성-개념.md)

## Prometheus — Pull 모델

### Push vs Pull

```
Push (StatsD, Datadog agent):  앱 ──메트릭 전송──▶ 수집 서버   "내가 보낸다"
Pull (Prometheus):  Prometheus ──HTTP GET /metrics──▶ 앱       "내가 긁어온다(scrape)"
```

각 타깃은 `/metrics`에 현재 값을 텍스트로 노출만 한다:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 152349
node_memory_used_bytes 8234123456
```

Prometheus가 `scrape_interval`(예: 15s)마다 이 URL을 GET해 파싱·저장.

### 왜 Pull인가 — 4가지 실질 이유

1. **Scrape 자체가 헬스체크** — 응답 없음 = 타깃 다운. 자동으로 `up{job="api"} 0` 기록. Push였다면 "메트릭이 안 오는 게 죽어서인지 한가해서인지" 구분 불가.
2. **중앙에서 타깃 관리** — "무엇을 모니터링 중인가"가 설정 한 곳에. Push는 누가 보내는지 서버가 모름(좀비 인스턴스).
3. **타깃이 모니터링 시스템을 몰라도 됨** — 앱은 `/metrics`만 열면 끝. 결합도↓.
4. **과부하 제어 용이** — 부하 크면 Prometheus가 주기를 늘림. Push는 트래픽 폭증 시 앱들이 서버를 DDoS.

### Service Discovery — Pull의 짝꿍

Pull의 약점은 "긁을 대상 목록을 누가 주냐". 오토스케일로 5→50대가 되면 수동 등록 불가 → **SD가 타깃 목록을 동적으로** 받아온다.

```yaml
scrape_configs:
  - job_name: 'api'
    kubernetes_sd_configs:      # K8s API에 "api 파드 다 줘"
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: api
        action: keep
```

K8s/Consul/EC2/DNS에서 "지금 살아있는 인스턴스 목록"을 실시간 반영. 파드가 뜨면 자동 추가, 죽으면 제거. **수동 등록 0** — Prometheus가 K8s 표준 모니터링이 된 이유.

### Pull의 예외 — Pushgateway

Pull이 안 통하는 경우: **수명이 짧아 scrape 타이밍을 못 맞추는 배치 잡**(새벽 3시 30초 돌고 끝나는 cron). 끝나기 전 Pushgateway에 push → Prometheus는 Pushgateway를 평소처럼 pull.

> ⚠️ **상시 서비스에 Pushgateway는 안티패턴**: 인스턴스가 죽어도 마지막 값이 stale하게 영원히 남고(`up` 헬스체크 무력화), 여러 인스턴스가 서로 덮어쓰며, SPOF가 된다. **오직 짧게 살고 죽는 배치 잡에만.**

## Exporter 생태계

세상 모든 게 `/metrics`를 내보내진 않는다(커널, 서드파티 DB, 레거시 앱). **Exporter = 대상의 네이티브 인터페이스를 읽어 Prometheus 포맷으로 재노출하는 번역기**.

```
[PostgreSQL] ──pg_stats(SQL)──▶ [postgres_exporter] ──/metrics──▶ [Prometheus]
```

### 핵심 3대장

- **node_exporter** — 호스트(머신) CPU/메모리/디스크/네트워크. 서버마다 하나. 인프라 모니터링 기본.
- **cAdvisor** — **컨테이너별** CPU/메모리. Docker/K8s 필수. node가 "호스트 전체"면 cAdvisor는 "컨테이너 단위로 쪼갠" 사용량.
- **blackbox_exporter** — **밖에서 사용자처럼 프로빙**. HTTP 응답코드/지연, TLS 만료일, TCP/DNS/ping.

> **whitebox vs blackbox**: whitebox(node/cAdvisor) = "엔진 내부 계기판"(내가 내 상태를 안다). blackbox = "밖에서 차 굴려보기"(사용자가 겪는 걸 본다). 내부는 멀쩡한데 방화벽 때문에 사용자가 못 들어오는 상황은 blackbox만 잡는다.

### 직접 계측 vs Exporter

```
내 앱        →  클라이언트 라이브러리로 직접 /metrics (비즈니스 메트릭: 주문 수 등)
호스트       →  node_exporter
컨테이너     →  cAdvisor
서드파티 DB  →  postgres/redis/..._exporter (코드 못 건드리니 번역기)
외형 점검    →  blackbox_exporter
짧은 배치잡  →  Pushgateway
```

```javascript
// Node.js — prom-client (직접 계측)
const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.05, 0.1, 0.3, 0.5, 1, 3],   // 우리 latency에 맞춘 버킷
});
```

> ⚠️ exporter의 `up`과 대상의 `up`은 다르다. postgres_exporter는 떠 있는데 PostgreSQL이 죽으면 → exporter가 던지는 `pg_up 0`으로 구분.

### 모니터링 범위 — 6계층

실무에선 스택을 층층이 덮는다:

```
6. Prometheus 자신  ← 메타 모니터링(별도 Prometheus가 감시)
5. 블랙박스(외형)    ← blackbox_exporter
4. 애플리케이션     ← 직접 계측 (RED + 비즈니스)
3. 미들웨어        ← postgres/redis/kafka_exporter
2. 컨테이너        ← cAdvisor, kube-state-metrics
1. 호스트          ← node_exporter
```

같은 `job`의 **인스턴스 50대면 50대 각각** `instance` 라벨로 분리 수집(SD가 자동 등록). 평소엔 `sum by(job)`으로 집계해 보다가, 문제 터지면 `by(instance)`로 쪼개 범인을 찾는다. 단 비용은 **활성 시계열 총개수** — 고카디널리티 라벨 금지(아래).

## PromQL

### 데이터 타입 — Instant vs Range Vector

메트릭 이름 하나 = **여러 시계열의 집합**(라벨 조합마다 하나).

- **Instant Vector**: 각 시계열의 **한 시점 값** — `http_requests_total`
- **Range Vector**: 각 시계열의 **시간 구간 값들** — `http_requests_total[5m]`

`rate()` 같은 함수는 **range vector를 요구**한다(변화율엔 여러 점이 필요). `[5m]`을 빼면 에러. (나머지: Scalar, String.)

### rate() vs irate()

둘 다 counter의 초당 증가율이지만 계산법이 다르다.

```
샘플:  10  12  18  19  50  51   (5분간)
rate  = (51-10)/300 ≈ 0.137/s   ← 구간 평균 (완만, 알림용)
irate = 마지막 두 샘플 기준        ← 순간값 (뾰족, 그래프용)
```

| | `rate()` | `irate()` |
|---|---------|-----------|
| 계산 | 구간 평균 | 마지막 2개 샘플 |
| 용도 | **알림, 장기 추세** | 빠르게 변하는 값 그래프 |
| 함정 | 짧은 스파이크 놓침 | 알림에 쓰면 노이즈 폭발 |

👉 **실무 기본은 `rate()`.**

> ⚠️ **rate의 숨은 능력 — counter 리셋 보정**: 앱 재시작 시 counter가 `152349→0`으로 떨어짐. 직접 빼면 음수. `rate()`/`irate()`는 **떨어지면 리셋으로 간주해 자동 보정**한다. 이게 counter를 직접 빼면 안 되고 rate로 감싸야 하는 이유.
> ⚠️ `[5m]` 구간은 **scrape 간격의 최소 4배**. 15s scrape에 `[15s]`면 샘플이 1~2개라 불안정.

### Aggregation — by / without

instant vector를 그룹으로 묶어 축소(50대를 하나로).

```promql
sum(rate(http_requests_total[5m]))                  # 전부 합침
sum by (job) (rate(http_requests_total[5m]))        # job별, instance 버림
sum without (instance) (rate(http_requests_total[5m])) # instance만 버림
```

- `by` = 남길 라벨, `without` = 버릴 라벨.
- 연산자: `sum`, `avg`, `max`, `min`, `count`, `topk`, `quantile`.
- ⚠️ **순서**: `sum(rate(...))` ✅ (rate 먼저, 집계 바깥). `rate(sum(...))` ❌ — sum은 instant vector라 rate가 받을 수 없음. **"rate 안쪽, 집계 바깥쪽"**.

### 실전 황금 쿼리

```promql
# 에러율(%) — RED의 E
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m])) * 100

# p95 지연 (Histogram) — RED의 D
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# 상위 5개 엔드포인트
topk(5, sum by (route) (rate(http_requests_total[5m])))
```

- `=~`는 정규식. 벡터끼리 나누면 **같은 라벨끼리 매칭**돼 나뉜다.
- ⚠️ p95에서 **`by(le)`를 빼면** histogram_quantile이 버킷을 못 알아봐 NaN. `le`는 반드시 남긴다.

### 자주 쓰는 함수

| 함수 | 용도 |
|------|------|
| `rate()` / `irate()` | counter 초당 증가율 |
| `increase()` | 구간 총 증가량("지난 1시간 에러 몇 건") |
| `histogram_quantile()` | 분위수 |
| `delta()` / `deriv()` | gauge 변화량/기울기 |
| `predict_linear()` | 선형 예측("디스크 4시간 뒤 꽉 참") |
| `absent()` | 메트릭 자체가 사라졌을 때 알림 |

> ⚠️ `increase()`는 내부적으로 rate×구간이라 보간 때문에 **정수가 살짝 안 떨어질 수 있음**(2건인데 2.03).

## 카디널리티 지옥

> Prometheus가 OOM으로 죽는 사고의 1순위. (개념 → [관찰가능성 개념](관찰가능성-개념.md))

**시계열 1개 = 메트릭 이름 + 라벨 조합 하나의 유니크.** 총 시계열 = **각 라벨 값 종류의 곱**.

```
method(2) × status(5) × instance(50) × route(20) = 10,000 시계열
+ user_id(100만)  = 100억 시계열  💥 → OOM → 모니터링 자체 다운
```

라벨 하나 추가는 **덧셈이 아니라 곱셈**.

```
❌ user_id, email, session_id, request_id, trace_id, ip, timestamp, full_url, 에러메시지
   → 값 종류가 무한정 늘어남
✅ method, status, route("/users/:id" 템플릿!), region, env, job, instance
   → 유한
```

> **판별 기준**: "이 라벨 값 종류가 시간이 지나며 무한정 늘어나는가?" 늘면 금지.
> **ID는 경로 템플릿으로 정규화** — `route="/users/:id"`(O) vs `path="/users/12345"`(X).

이건 버그가 아니라 **의도된 설계**다. Prometheus는 저카디널리티 집계 메트릭에 최적화됨. "유저별/요청별 디테일"은 처음부터 Prometheus 일이 아니라 **log(Loki)/trace(Tempo)의 몫** — "높은 카디널리티가 필요하면 metric이 아니라 log/trace로".

### 방어법

```promql
# 카디널리티 범인 찾기: 메트릭별 시계열 수 TOP 10
topk(10, count by (__name__)({__name__=~".+"}))
```

```yaml
# 이미 터졌으면 수집 단계에서 문제 라벨 drop
metric_relabel_configs:
  - source_labels: [user_id]
    action: labeldrop
    regex: user_id
```

scrape별 `sample_limit`으로 폭주 타깃 차단.

## Grafana

### 정체성 — 저장하지 않는다

```
[Prometheus]─┐
[Loki]       ─┼─▶ [Grafana] ─▶ 사람 눈   (쿼리 던지고 그림만)
[Tempo]      ─┘
```

Grafana는 자기 시계열 DB가 없다. 패널마다 **연결된 데이터소스에 쿼리(PromQL/LogQL)를 실시간으로 던지고** 결과를 렌더링. 진짜 가치는 **여러 소스를 한 화면에** 모으는 것.

### 구성 — Dashboard → Panel → Query

```
Dashboard (한 화면)
├─ Panel: "요청률"  ─ Query: sum(rate(http_requests_total[5m]))
├─ Panel: "에러율"  ─ Query: ...5xx 비율...
└─ Panel: "p95"     ─ Query: histogram_quantile(...)
```

시각화 타입: Time series(꺾은선), Stat(큰 숫자), Gauge, Table, **Heatmap**(latency 버킷 분포에 최적), Logs(Loki용). 패널 하나에 쿼리 여러 개 겹치기 가능.

### Variable($변수) — 대시보드 재사용

상단 드롭다운. 없으면 인스턴스 50대마다 대시보드 50개를 만들어야 한다.

```promql
rate(http_requests_total{instance="$instance", env="$env"}[5m])
```

드롭다운을 바꾸면 대시보드 전체가 갱신 → **한 대시보드로 모든 대상**.

| 종류 | 설명 |
|------|------|
| **Query** | 데이터소스 질의로 값 동적 생성: `label_values(node_cpu_seconds_total, instance)` |
| **Custom** | 고정 목록(`prod, staging, dev`) |
| **Interval** | 시간 구간(`$__interval`) |
| **Datasource** | 데이터소스 자체 전환 |

> `label_values()`는 인스턴스 목록을 **하드코딩 안 하고** Prometheus에 물어봄 → 오토스케일 시 드롭다운 자동 갱신(SD와 같은 철학). Multi-value는 `instance=~"$instance"`(정규식)로.

### Correlation — 데이터소스를 묶는 진짜 힘

- 메트릭 스파이크 클릭 → **그 시각 Loki 로그로 점프**(같은 라벨 자동 필터)
- 로그의 `trace_id` 클릭 → **Tempo 트레이스 폭포수로 점프**
- **Exemplar**: 메트릭 위 점으로 찍힌 샘플 trace 클릭 → 그 순간 느렸던 실제 요청

→ "metric 감지 → trace 위치 → log 원인" 릴레이를 한 화면에서 클릭 몇 번으로.

### Dashboard as Code

대시보드 = **JSON 모델**(패널 위치/쿼리/타입). UI에서 만들면 Grafana 내부 DB(기본 SQLite, 운영은 MySQL/Postgres)에만 저장된다.

```json
{
  "type": "timeseries",
  "datasource": { "uid": "prometheus-prod" },   // 어디서
  "targets": [{ "expr": "rate(http_requests_total[5m])" }], // 무슨 쿼리로
  "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 } // 위치
}
```

**JSON 직렬화가 되는 이유**: 패널이 데이터를 얻는 방법이 **`(datasource 참조 + 쿼리 문자열 + 위치)`로 환원**되기 때문. 임의 코드/엔드포인트 호출이 없고, 복잡한 가공은 전부 **PromQL 문자열 안으로** 캡슐화. "어떻게 가져오냐(HTTP/파싱)"는 데이터소스 플러그인이 숨김. → 위젯이 순수 선언형이 되고 → Git 버전관리 가능.

| 워크플로우 | 저장 | 버전관리 |
|-----------|------|---------|
| **UI-first** | Grafana DB 안에만 | ❌ Git 밖, 컨테이너 날아가면 소실 |
| **Code-first**(provisioning) | Git → 배포 시 주입 | ✅ diff·리뷰·롤백 |

> 12-Factor의 "설정을 코드로"([12/15-Factor App](../아키텍처/12-15-Factor-App.md))가 모니터링에도 그대로. 중요한 대시보드는 export해 Git에.

## Loki — 로그계의 Prometheus

### 발상 전환 — 라벨만 인덱싱

ELK(Elasticsearch)는 로그 **본문 전체를 역색인**([Elasticsearch](../DB/Elasticsearch.md)) → 아무 단어나 빠르지만 인덱스가 원본보다 클 때도 있어 **비용 폭발·운영 복잡**.

Loki의 발상: "대부분의 디버깅은 '어느 앱/어느 시간대'로 좁힌 뒤 grep인데, 전부 검색 가능하게 만들 필요가 있나?"

```
{app="api", env="prod", level="error"}   ← 라벨 (이것만 인덱싱, 작고 가벼움)
2026-06-10 03:14  user 12345 failed login timeout   ← 본문 (인덱싱 X, gzip 압축 → S3)
```

본문 검색은 **라벨로 청크를 좁힌 뒤 brute-force grep**:

```
{app="api", level="error"} |= "timeout"
└─ ① 인덱스로 청크 위치(전체 1%로 축소) ─┘ └─ ② 남은 청크만 병렬 스캔 ─┘
```

ELK처럼 "아무 단어나 즉각"은 아니지만, 라벨로 잘 좁히면 충분히 빠르고 **인덱스 비용이 없어 훨씬 쌈**.

### Prometheus와 "같은" 시스템인 이유

Loki는 **Prometheus의 라벨 데이터 모델을 그대로** 가져왔다.

```
Prometheus:  http_requests_total{app="api", env="prod"}
Loki:                           {app="api", env="prod"}   ← 똑같은 라벨!
```

→ 같은 라벨이니 metric ↔ log를 즉시 점프(Grafana correlation의 실체).

### ELK vs Loki

| | ELK | Loki |
|---|-----|------|
| 인덱싱 | 본문 전체 역색인 | **라벨만** |
| 임의 단어 검색 | 빠름, 복합 쿼리 강력 | 라벨로 좁힌 뒤 grep |
| 저장 비용 | 비쌈 | **쌈**(S3 + 압축) |
| 운영 난이도 | 높음(샤드/힙) | 낮음 |
| 메트릭 연동 | 별개 | **Prometheus 라벨 공유** |

👉 "임의 키워드 전문 검색·복잡 분석" → ELK. "라벨로 좁혀 디버깅 + Prometheus와 한 세트 + 비용 절감" → Loki.

### 수집 파이프라인 — 로그파일의 종착지

Loki는 로그파일을 없애는 게 아니라, **흩어진 로그파일을 한 곳에 모으는** 백엔드다. 앱은 여전히 stdout/파일로 뱉고, **에이전트**가 중간에서 수집한다.

```
[앱] ──stdout/파일──▶ [Promtail / Grafana Alloy] ──push──▶ [Loki]
                       tail해서 라벨 붙여 전송              라벨 인덱스 + S3 청크
```

- **Promtail / Grafana Alloy**: 각 노드에서 로그를 tail → 라벨(`app`,`env`,`pod`) 붙여 Loki로 **push**.
- ⚠️ Loki는 **push로 받는다**(Prometheus pull과 반대). 로그는 발생 즉시 흘려보내는 이벤트 스트림.
- 효과: 파드가 죽어도 Loki에 로그가 남고, 50대든 500대든 SSH 없이 한 화면에서 조회.

> "DB냐?" → 저장·조회하니 DB 성격은 맞지만 범용 DB는 아님. **시간+라벨로 append-only 되는 로그에 특화된 저장소**. 메트릭에 Prometheus가 있듯, 로그엔 Loki.

## LogQL

PromQL을 로그용으로 변형. **"라벨로 스트림 고르고 → 파이프로 거른다"**.

```logql
{app="api", env="prod"}  |= "error"  | json  | status >= 500
└─ ① 스트림 셀렉터(필수) ┘ └────── ② 파이프라인(선택) ──────┘
```

### ① Stream Selector (필수)

```logql
{app="api", env="prod"}      # AND
{app=~"api|web"}             # 정규식
{app="api", level!="debug"}  # 제외
```

`{}` 없이는 쿼리 불가. 여기서 읽을 청크를 좁힌다 — 잘 좁혀야 빠름.

### ② 파이프라인 — 4종류

```logql
# (a) 라인 필터 (grep)
{app="api"} |= "error"        # 포함
{app="api"} != "healthcheck"  # 제외
{app="api"} |~ "5\\d\\d"      # 정규식 매칭
{app="api"} !~ "DEBUG|TRACE"  # 정규식 제외

# (b) 파서 — 구조화 필드 추출
{app="api"} | json            # JSON → 필드를 라벨처럼
{app="api"} | logfmt          # key=value

# (c) 라벨 필터 — 추출 필드로 거르기
{app="api"} | json | status >= 500 and latency > 1.0
{app="api"} | json | duration > 200ms   # 단위 인식

# (d) 라인 포맷
{app="api"} | json | line_format "{{.method}} {{.status}}"
```

| 연산자 | 의미 |
|--------|------|
| `\|=` / `!=` | 포함 / 미포함 |
| `\|~` / `!~` | 정규식 매칭 / 불매칭 |

> ⚠️ **성능**: `|=`(문자열)가 `|~`(정규식)보다 빠름. **싼 필터를 먼저** 걸어 후보를 줄인 뒤 파싱.
> ⚠️ **카디널리티 우회 핵심**: `user_id` 같은 고카디널리티는 **라벨이 아니라 본문에** 두고 `| json | user_id="12345"`로 쿼리 시점에 필터. ("식별자는 라벨이 아니라 본문에")

### 킬러 기능 — 로그를 메트릭으로

Loki가 "로그계의 Prometheus"인 진짜 이유. **로그를 숫자로 변환**해 그래프·알림을 만든다.

```logql
rate({app="api"} |= "error" [5m])               # 에러 로그 초당 발생률
count_over_time({app="api"} |~ "5\\d\\d" [5m])  # 5분간 5xx 개수
avg_over_time({app="api"} | json | unwrap latency [5m]) by (route)  # 평균 latency
```

`rate()`, `count_over_time()`, `sum by()` — PromQL과 같은 문법. **메트릭이 없는 값도 로그만 있으면** 메트릭처럼 본다(앱이 `payment_failed` 로그만 남겨도 실패율 알림 생성).

> ⚠️ 로그 기반 메트릭은 매 쿼리마다 청크를 스캔하므로, 자주 보거나 알림 거는 핵심 지표는 **앱에서 진짜 Prometheus 메트릭으로** 내보내는 게 효율적. 로그 메트릭은 보조.

```
Log query    → 로그 라인 그대로     {app="api"} |= "error"
Metric query → 로그를 숫자로 집계    rate({app="api"} |= "error" [5m])
```

## 알림 — Alert Rule + Alertmanager

> 알림 철학(symptom-based, SLO/burn rate)은 → [관찰가능성 개념](관찰가능성-개념.md#알림-철학--alert-fatigue-방지)

**두 컴포넌트가 역할 분리**:

```
[Prometheus] "조건 평가해 alert 발화"(언제) ──▶ [Alertmanager] "그룹핑/라우팅/억제해 전송"(어떻게/누구에게)
```

왜 분리? 조건 판단(데이터)과 알림 전달(전송 정책)은 다른 관심사. 여러 Prometheus(샤딩) → 하나의 Alertmanager로 모아야 중복 제거·라우팅이 일관됨.

### ① Alert Rule (Prometheus 측)

```yaml
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m                       # 지속 시간 ⚠️
        labels: { severity: critical } # 라우팅용
        annotations:                   # 사람용
          summary: "에러율 5% 초과 ({{ $value | humanizePercentage }})"
```

**생명주기 — `for`가 핵심**:

```
조건 거짓        → inactive
조건 참(방금)    → pending   ← for 동안 머무름
조건 참 5분 지속 → firing    ← 그제서야 Alertmanager로 전송!
```

- `for: 5m` = 5분 연속 참이어야 발화. 순간 스파이크 무시 → **알림 피로 1차 방어선**.
- `labels` = 라우팅·그룹핑 키. `annotations` = 사람용 메시지(runbook 링크), 라우팅엔 안 쓰임.

### ② Alertmanager — 4가지 처리

```yaml
route:
  group_by: ['alertname', 'cluster']  # (a) Grouping: 50대 동시 다운 → 1개 메시지
  group_wait: 30s
  repeat_interval: 4h
  receiver: 'default'
  routes:                              # (b) Routing: 라벨 보고 채널 분기
    - matchers: [ severity="critical" ]
      receiver: 'pagerduty'            # 전화로 깨움
    - matchers: [ team="payments" ]
      receiver: 'slack-payments'

inhibit_rules:                         # (c) Inhibition: 상위 알림이 하위 억제
  - source_matchers: [ alertname="ClusterDown" ]
    target_matchers: [ severity="warning" ]
    equal: ['cluster']
```

- **(a) Grouping**: 비슷한 알림 묶기. 노드 50개 다운 → 50개가 아니라 1개로.
- **(b) Routing**: 라벨로 트리 매칭 → 채널(receiver) 결정.
- **(c) Inhibition**: "클러스터 다운(critical)"이 떴으면 그 안의 "파드 응답없음(warning)" 100개를 억제 → 근본 원인 하나만.
- **(d) Silence**: 점검·배포 시간 동안 특정 알림 수동 음소거(Alertmanager UI).

```
Prometheus 조건 평가 → (for 지속) FIRING → Alertmanager
  → Inhibition → Grouping → Routing → Silence → Receiver(Slack/PagerDuty/Email/Webhook)
```

### 실무 핵심

- **알림은 사용자 영향(RED/SLO)에만** — "CPU 90%"로 깨우지 말고 "에러율/p99 SLO 위반"으로. 자원은 대시보드, 알림은 증상.
- **모든 alert에 runbook 링크**(annotations) — 새벽 3시에 깨운 사람이 뭘 할지 바로 알도록.
- **severity는 2~3단계로** — critical(즉시 깨움)/warning(업무시간). 잘게 나누면 라우팅 지옥.

## 운영 아키텍처 — 저장소는 어떻게 관리하나

### 자체 저장 엔진 내장 — 별도 DBMS 불필요

셋 다 **저장 로직을 프레임워크 안에 내장**한다. PostgreSQL/MongoDB 같은 외부 DBMS를 따로 띄워 붙일 필요가 없다.

```
❌ 별도 DBMS 제품 연결      → 불필요 (각자 저장 엔진 내장)
✅ 데이터를 얹을 저장 매체   → 필요 (디스크 또는 S3 버킷)
```

- **Prometheus** — 로컬 디스크에 **자체 TSDB 내장**. 바이너리 하나 띄우면 별도 연결 0으로 동작. 장기보관·HA가 필요할 때만 `remote_write`로 Mimir/S3를 *추가*(필수 아님).
- **Loki / Tempo** — 내장 엔진 + 스토리지 선택. **소규모**는 로컬 파일시스템으로 단일 바이너리 즉시 실행, **스케일**하면 object storage(S3/GCS/**MinIO**) 버킷 연결. 여기서 S3는 **DB가 아니라 압축 청크 보관함**(쿼리·인덱싱 로직은 Loki/Tempo 안에).
- **Grafana** — 저장 안 함. 다만 **쿼리 대상**으로 데이터소스 URL을 등록("DB 연결"이 아니라 쿼리 라우팅 설정).

> 그래서 docker-compose 한 장에 Prometheus + Loki + Grafana를 띄우면 **외부 DB 없이 로컬 볼륨만으로** 바로 돌아간다.

### 왜 저장소를 3개로 나눴나 — 데이터 특성이 다름

| 신호 | 데이터 모양 | 필요한 최적화 |
|------|-----------|-------------|
| Metric | 고빈도 작은 숫자 | 시계열 델타 압축(수천 배) |
| Log | 대용량 비정형 텍스트 | 압축 + 라벨 인덱스 |
| Trace | span 트리(부모-자식) | trace_id 조회 |

하나의 DB로 셋을 다 하면 뭐 하나는 반드시 비효율(메트릭 엔진에 텍스트를 밀면 압축 안 먹고, 로그 엔진으로 분위수 집계는 느림). **분리 = 각자 자기 데이터에 최적화**하려는 의도.

### "DB 3개 따로"의 부담을 줄이는 공통 인프라

전통적 DB 3개(PostgreSQL+MongoDB+ES 각자 튜닝)와 달리, Loki·Tempo·Mimir는 **같은 아키텍처 패턴**을 공유한다:

1. **저장은 전부 S3 한 곳** — 셋 다 "라벨 인덱스(작음) + 압축 청크를 S3에". 무거운 stateful 상태를 S3가 통째로 받아 백업·확장·내구성 일괄 해결.
2. **컴포넌트가 stateless** — 쿼리/수집 프로세스는 상태가 없어 늘리고 줄이면 됨.
3. **한 제품군이라 운영 일관** — Helm chart로 한 번에 배포, 같은 라벨 모델.
4. **수집도 하나로 통합** — **Grafana Alloy**(또는 OTel Collector) 에이전트 하나가 metric/log/trace를 다 수집해 분배.

→ 체감은 "DB 3개 운영"보다 **"S3 + stateless 쿼리 엔진 3종 + 에이전트 1개"**.

### 부담되면 두 출구

- **매니지드**: Grafana Cloud, AWS Managed Prometheus/Grafana — 저장·확장·업그레이드 위임.
- **단일 백엔드 올인원**: **SigNoz / OpenObserve**(ClickHouse 하나에 metric/log/trace 다 얹음), Datadog/Elastic. **통합 = 관리 단순 ↔ 분리 = 각자 최적화·각자 스케일**의 트레이드오프.

### Mimir vs Prometheus — 경쟁 아님

```
[Prometheus] ──remote_write──▶ [Mimir / Thanos / VictoriaMetrics]
  여전히 scrape·수집 담당          장기 저장 + 수평 확장 + 멀티테넌트
```

- 수집·쿼리의 사실상 표준은 **Prometheus(+PromQL)**. 단일 Prometheus는 로컬 보관 ~15일, ~수백만 시계열이 한계.
- **Mimir는 Prometheus를 대체하지 않고 뒤에 붙인다** — `remote_write`를 받아 S3 장기보관 + 수평확장 + 멀티테넌트. **PromQL·데이터 모델을 그대로** 씀.
- ⚠️ "LGTM의 M = Mimir"는 **Grafana Labs 자사 제품 브랜딩**일 뿐, **실제 현장에서 M 자리는 거의 다 Prometheus**. "Prometheus냐 Mimir냐"가 아니라 "Prometheus 단독이냐, Prometheus + Mimir냐". 같은 자리 대안: **Thanos**(사이드카), **VictoriaMetrics**(경량·고속).

| 상황 | 선택 |
|------|------|
| 단일/소규모, 보관 2주~한두 달 | **Prometheus 단독** (대다수) |
| 장기 보관·HA 이중화 | + Thanos / Mimir |
| 수백만+ 시계열, 수평 확장, 멀티테넌트 | + Mimir / VictoriaMetrics |

## 핵심 질의응답

**Q. Prometheus로 보통 어디까지 모니터링하나? 모든 인스턴스 다?**
A. 6계층(호스트→컨테이너→미들웨어→앱→블랙박스→Prometheus 자신)을 덮고, 같은 종류의 **모든 인스턴스를 각각** 수집한다. 단 Service Discovery가 자동 등록하므로 손이 안 가고, 비용은 인스턴스 수가 아니라 **활성 시계열 총개수** — 고카디널리티만 피하면 다 긁어도 된다.

**Q. Loki는 로그파일 대신 쓰는 별도 DB인가?**
A. 로그파일을 없애는 게 아니라, 흩어진 로그파일(stdout 포함)을 에이전트(Promtail/Alloy)가 수집해 push하면 라벨 인덱스+S3 청크로 저장하는 **로그 전용 백엔드**. 로그파일들의 종착지. 파드가 죽어도 남고 중앙에서 조회된다.

**Q. Grafana 대시보드가 JSON으로 표현되는 이유는?**
A. 패널이 데이터를 얻는 방법이 `(datasource 참조 + 쿼리 문자열 + 위치)`로 환원되기 때문. 임의 코드/엔드포인트 호출 없이 복잡성을 PromQL 문자열 안으로 캡슐화해서 순수 선언형이 된다. 그래서 Git 버전관리가 가능(단 UI-first면 DB에 갇혀 추적 불가, code-first/provisioning이어야 함).

**Q. rate를 안쪽에 둬야 하는 이유?**
A. `sum`은 instant vector를 내놓는데 `rate`는 range vector(`[5m]`)를 입력으로 요구한다. `rate(sum(...))`은 타입이 안 맞아 에러. 항상 `sum(rate(...))`.

**Q. Pushgateway는 언제 쓰나?**
A. 수명이 짧아 scrape 타이밍을 못 맞추는 배치 잡에만. 상시 서비스에 쓰면 stale 값이 영원히 남고 SPOF가 되는 안티패턴.

**Q. Loki/Tempo/Prometheus 다 별도 저장소면 DB가 너무 많아 관리가 어렵지 않나?**
A. 별도 저장소는 맞지만(데이터 특성이 달라 분리가 의도된 설계), 전통적 DB 3개와 달리 **공통 인프라로 묶인다** — 무거운 상태는 전부 S3 한 곳에, 컴포넌트는 stateless, 한 제품군이라 배포 일관, 수집은 Alloy 에이전트 하나로 통합. 체감은 "S3 + stateless 쿼리 엔진 3종 + 에이전트 1개". 부담되면 매니지드(Grafana Cloud)나 올인원(SigNoz)으로.

**Q. 별도 저장소를 연결할 필요 없이 프레임워크가 자체 제공하나?**
A. 외부 DBMS(PostgreSQL 등)는 불필요 — 셋 다 저장 엔진을 내장한다. Prometheus는 로컬 디스크로 즉시 동작, Loki/Tempo는 소규모면 로컬 FS·커지면 S3 버킷만 연결(S3는 DB가 아니라 파일 보관함). Grafana만 쿼리 대상으로 데이터소스 URL을 등록.

**Q. Mimir와 Prometheus 중 뭘 더 많이 쓰나?**
A. 압도적으로 Prometheus. 둘은 경쟁이 아니라 Mimir가 Prometheus 뒤에 붙는 장기저장·수평확장 레이어다(PromQL 그대로). "LGTM의 M=Mimir"는 Grafana Labs 브랜딩일 뿐 실제 M 자리는 대부분 Prometheus. 규모가 커질 때만 Thanos/Mimir/VictoriaMetrics를 추가.

## 주의사항 / 자주 하는 실수

- **counter를 직접 빼기 ❌** — 재시작 리셋 시 음수. 항상 `rate()`로 감싼다(리셋 자동 보정).
- **`rate(sum(...))` ❌** — rate는 range vector 필요. `sum(rate(...))`.
- **histogram_quantile에서 `by(le)` 누락 ❌** — 버킷 인식 실패로 NaN.
- **고카디널리티 라벨(user_id 등) ❌** — Prometheus도 Loki도 OOM. 식별자는 본문/trace로.
- **Pushgateway를 상시 서비스에 ❌** — stale 값 영구 잔존, SPOF.
- **`|~`(정규식) 남발 ❌** — `|=`로 먼저 좁히고 정규식은 나중에.
- **UI로만 대시보드 제작 ❌** — Grafana DB에만 남아 소실. JSON export → Git.
- **알림 `for` 생략 ❌** — 스파이크마다 울림.

## 참고

- [관찰가능성 개념](관찰가능성-개념.md) ← 3축/메트릭 타입/RED·USE/알림 철학
- [Elasticsearch 검색 엔진](../DB/Elasticsearch.md) ← Loki와 대비되는 역색인
- [12/15-Factor App](../아키텍처/12-15-Factor-App.md) ← XI. Logs
- [Prometheus 공식 — Querying](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Loki — LogQL](https://grafana.com/docs/loki/latest/query/)
- [Alertmanager 공식](https://prometheus.io/docs/alerting/latest/alertmanager/)
