# kis-trader — Architecture Plan v2 (Claude)

> 퀀트 알고리즘 백테스팅 + 시뮬레이션/실전 자동매매 웹서비스
> Local deployment: Windows PC + Mac Mini
> Date: 2026-03-27
> 실전 매매 서비스는 내부망 전용 분리 운영
> v2.1: Gemini 리뷰 반영 — Redis/Celery 큐, Docker 샌드박스, KIS 토큰 데몬 추가

---

## 1. 핵심 목표

1. 퀀트 알고리즘 파라미터를 웹 UI에서 설정하고 과거 데이터로 백테스팅
2. 백테스트 히스토리를 비교·분석해 최적 전략 선택
3. 선택 전략을 KIS API 시뮬레이션 계좌로 페이퍼 트레이딩 검증
4. 검증된 전략을 KIS API 실계좌 자동매매에 적용 (ADMIN 전용)
5. Slack 알림 + 일일 리포트 + 이상 감지 자동 처리

---

## 2. 사용자 권한 구조

```
ADMIN (본인 1명)
  ├── 실계좌 KIS API 키 등록/관리
  ├── 실전 자동매매 활성화/중단
  ├── 실계좌 수동 주문
  ├── 전체 유저 관리
  └── 감사 로그 조회

USER (일반 로그인 유저)
  ├── 전략 생성/수정/삭제 (본인 것만)
  ├── 백테스팅 실행 및 결과 조회
  ├── 시뮬레이션 계좌 자동매매
  └── 본인 데이터만 조회/수정
```

---

## 3. 보안 설계

### 3-1. 인증 (Authentication)

| 항목 | 내용 |
|---|---|
| Access Token | JWT, **15분** 만료 |
| Refresh Token | 7일, DB 저장 (서버에서 강제 만료 가능) |
| 로그인 실패 | 5회 연속 실패 → 계정 잠금 + Slack 알림 |
| 동시 세션 | ADMIN: 1개만 허용, USER: 3개까지 |
| 비밀번호 | bcrypt 해싱, 최소 12자 + 특수문자 강제 |

### 3-2. 실전 매매 전용 보호

| 레이어 | 내용 |
|---|---|
| 역할 검사 | ADMIN 역할 필수 (모든 실전 매매 API) |
| **2차 인증 (TOTP)** | 실전 매매 API 호출 시 Google Authenticator OTP 추가 요구 |
| **IP 화이트리스트** | ADMIN 계정은 사전 등록 IP에서만 로그인 허용 |
| 주문 확인 토큰 | 주문 요청 시 서버 발급 단기 토큰(60초) 포함 → CSRF 방어 |
| 주문 한도 | 단일 주문 최대 금액 설정 (초과 시 차단) |

### 3-3. KIS API 키 보호

- Fernet 대칭 암호화 후 DB 저장 (복호화 키는 환경변수 `KIS_ENCRYPT_KEY`)
- API 키 응답 시 평문 절대 노출 금지 → `****1234` 마스킹
- 키 변경 시 기존 활성 전략 자동 중단 + Slack 알림

### 3-4. 감사 로그 (Audit Log)

모든 중요 액션을 `audit_logs` 테이블에 기록:
- 실전 주문 실행 (who / when / what / amount)
- 전략 활성화/중단
- KIS 키 등록/변경
- 로그인 성공/실패
- 계정 잠금/해제

### 3-5. 프론트엔드 보안

- 실전 매매 UI: ADMIN만 렌더링 (서버사이드 권한 체크 + 클라이언트 숨김 병행)
- 실전 주문 버튼: 2단계 확인 모달 (주문 금액 직접 입력으로 재확인)
- HTTPS 전용 (Cloudflare Tunnel → 평문 내부 통신은 LAN 한정)

---

## 4. 데이터 전략

### 주식 데이터 — KIS API 미사용, 외부 소스

| 용도 | 소스 | 이유 |
|---|---|---|
| 초기 벌크 수집 (일봉 전종목 3년+) | **pykrx** | KRX 공식, API키 불필요, 무제한 |
| 일별 cron 업데이트 | **pykrx** | 동일 소스 유지 |
| 재무 데이터 (PER, PBR, ROE) | **pykrx** / 공공데이터포털 | 팩터 스크리닝용 |
| 수정주가 (배당/분할 반영) | **pykrx** 기본 제공 | 백테스트 정확도 |

### KIS API — 매매 전용

| 용도 | API |
|---|---|
| 시뮬레이션 주문 | KIS 가상계좌 REST |
| 실계좌 주문 | KIS 실계좌 REST |
| 잔고/체결 조회 | KIS REST |
| 실시간 체결 알림 | KIS WebSocket (실전 매매 Phase) |

> KIS REST API는 macOS에서도 동작. WebSocket 실시간은 실전 매매 Phase에서 테스트 후 결정.

### 매매 시점

```
일봉 전략 (대부분의 퀀트 전략):
  T일 장 마감 후 cron → 신호 계산 → T+1일 시가 주문 실행

분봉/실시간 전략 (Phase 5 이후):
  KIS WebSocket → 실시간 호가 수신 → 즉시 주문
```

---

## 5. 시스템 아키텍처

```
[ 외부 인터넷 ]
     ↓ HTTPS (Cloudflare Tunnel)
┌─────────────────────────────────────────┐
│ Windows PC — 외부 노출 영역              │
│  ├── Next.js Frontend          :3000    │  ← 백테스팅/시뮬 UI
│  ├── FastAPI Backend            :8000   │  ← 백테스팅/시뮬 API
│  ├── Redis                      :6379   │  ← 태스크 큐 (내부망 전용)
│  ├── PostgreSQL                 :5432   │
│  ├── Data Collector (cron)              │  ← pykrx + APScheduler
│  ├── KIS Token Refresher (daemon)       │  ← 24h 토큰 자동 갱신
│  ├── Slack Bot Worker                   │  ← 알림/리포트
│  └── cloudflared                        │
└─────────────────────────────────────────┘
          │ Redis 큐 (내부망)
┌─────────────────────────────────────────┐
│ Mac Mini — Celery Worker                │
│  ├── Backtest Worker (Celery)           │  ← CPU 헤비 태스크 소비
│  └── Docker daemon                      │  ← 커스텀 코드 샌드박스 실행
└─────────────────────────────────────────┘
          │ (별도, 내부망 전용)
┌─────────────────────────────────────────┐
│ Windows PC — 내부망 전용 영역            │
│  └── Real Trading Service       :8002   │  ← 실계좌 전용, 인터넷 미노출
└─────────────────────────────────────────┘
          │ REST API (매매 전용)
┌─────────────────────────────────────────┐
│ KIS 한국투자증권                          │
│  └── openapi.koreainvestment.com        │
└─────────────────────────────────────────┘
```

### 네트워크 접근 정책

| 서비스 | 외부(인터넷) | 내부망 |
|---|---|---|
| Frontend :3000 | O (Cloudflare) | O |
| API Backend :8000 | O (Cloudflare) | O |
| Real Trading Service :8002 | **X 차단** | O (localhost만) |
| Backtest Worker :8001 | **X 차단** | O |
| PostgreSQL :5432 | **X 차단** | O |

### Mac Mini 태스크 큐 (Redis + Celery)

- FastAPI Backend가 백테스트 요청을 Redis 큐에 push
- Mac Mini Celery Worker가 큐에서 태스크 소비 → 실행 → 결과를 Redis에 저장
- 파라미터 최적화처럼 5~10분 걸리는 작업도 HTTP timeout 없이 안정적 처리
- Mac Mini 오프라인 시 큐에 대기 → 복구 후 자동 처리
- Frontend는 WebSocket으로 진행률 스트리밍

### 커스텀 코드 샌드박스 (Docker 컨테이너)

- 사용자 Python 코드 → Mac Mini에서 에피머럴 Docker 컨테이너로 실행
- 컨테이너 옵션: `--network none`, `--memory 512m`, `--cpus 1`, `--read-only`
- 실행 완료 즉시 컨테이너 삭제

### 실전 매매 서비스 분리 효과

- KIS API 키/실계좌 서비스가 인터넷에 완전 미노출 → 물리적 격리
- Cloudflare 또는 공개 API가 침해당해도 실계좌 무영향
- TOTP 등 복잡한 보안 레이어 없이 내부망 접근 자체가 보안
- 실전 매매 UI는 `localhost:8002` 에서 직접 접근

---

## 6. 기술 스택

| 레이어 | 기술 |
|---|---|
| Frontend | Next.js 15 + TypeScript + my-ui-lib |
| 차트 | TradingView Lightweight Charts |
| API Backend | FastAPI 0.115 + SQLAlchemy 2.0 + Alembic |
| 인증 | JWT (python-jose) + TOTP (pyotp) |
| Backtest Worker | FastAPI + pandas 2.x + pandas-ta + vectorbt |
| Real Trading Service | FastAPI (내부망 전용, KIS API 연동) |
| 태스크 큐 | **Redis + Celery (or ARQ)** — Mac Mini 비동기 작업 큐 |
| 데이터 수집 | pykrx + APScheduler |
| 알림 | Slack SDK (slack_sdk) |
| 커스텀 코드 실행 | **에피머럴 Docker 컨테이너** (네트워크 차단, CPU/메모리 제한) |
| TimescaleDB | PostgreSQL 17 확장 — OHLCV 시계열 최적화 (선택적 적용) |
| 실시간 | FastAPI WebSocket |
| DB | PostgreSQL 17 |
| 암호화 | cryptography (Fernet) |
| 인프라 | Docker Compose + Cloudflare Tunnel |

---

## 7. 지원 알고리즘 (웹 UI에서 파라미터 설정)

| 타입 | 이름 | 핵심 파라미터 | 추천값 |
|---|---|---|---|
| `MA_CROSS` | 이동평균 교차 | short_period, long_period, ma_type | 5/20, SMA |
| `RSI` | RSI 과매수/과매도 | period, oversold, overbought | 14, 30, 70 |
| `MACD` | MACD 시그널 교차 | fast, slow, signal | 12, 26, 9 |
| `BOLLINGER` | 볼린저밴드 | period, std_dev, mode | 20, 2.0, reversion |
| `MOMENTUM` | 모멘텀/ROC | period, threshold | 20, 0 |
| `STOCHASTIC` | 스토캐스틱 | k_period, d_period, smooth | 14, 3, 3 |
| `MEAN_REVERT` | 평균회귀 (Z-Score) | lookback, z_score_threshold | 20, 2.0 |
| `FACTOR` | 팩터 투자 | factors[], weights[] | PBR+ROE 조합 |
| `MULTI` | 다중 조건 AND/OR | conditions[], operator | — |
| `CUSTOM` | Python 코드 직접 입력 | code (str) | — |

### 공통 매매 파라미터 (UI에서 설정, 추천값 표시)

```
초기 투자금         default: 10,000,000원
포지션 사이즈       고정 비율 / 켈리 공식 / 변동성 기반 선택
스탑로스            default: 3%
익절                default: 10%
수수료              default: 0.015%
거래세              default: 0.2% (매도 시)
슬리피지            default: 0.1% (보수적 가정)
최대 보유 종목 수   default: 5
```

---

## 8. 종목 스크리닝 (웹 UI)

### 스크리닝 조건

```
섹터 필터        KOSPI 업종 (반도체, 바이오, 금융, 자동차, ...)
시가총액         대형주(>1조) / 중형주 / 소형주
거래량           최소 일평균 거래량
PER              범위 설정 (예: 5~20배)
PBR              범위 설정 (예: 0.5~2배)
ROE              최소값 (예: 10% 이상)
52주 대비        고점 대비 하락폭 / 저점 대비 상승폭
변동성(ATR)      낮음 / 보통 / 높음
```

### 빠른 선택 프리셋

- 코스피 대형주 TOP 50
- 섹터별 전체 종목
- 직접 종목 입력 (멀티 셀렉트)
- 즐겨찾기 종목 리스트

> 백테스팅 종목 수 최대 30개 권장, 초과 시 경고

---

## 9. 백테스팅 품질 옵션

| 옵션 | 설명 |
|---|---|
| **단순 백테스팅** | 전체 기간 한 번에 실행 |
| **Walk-Forward** | 기간 분할 (train/test) → 과적합 방지 |
| **파라미터 최적화** | 그리드 서치로 최적 파라미터 탐색 (과적합 경고 표시) |
| **전략 비교** | 동일 조건으로 N개 전략 동시 실행 → equity curve 오버레이 |

### 정확도 설정

```
수정주가 사용        default: ON  (배당/분할 반영)
Survivor Bias 경고   상장폐지 종목 미포함 시 경고 표시
거래 비용 반영       수수료 + 거래세 + 슬리피지 (개별 설정)
벤치마크 비교        코스피 지수 수익률 병렬 표시
```

---

## 10. 백테스팅 결과 분석

### 성과 지표

```
총 수익률 / 연환산 수익률 / 코스피 대비 알파
최대 낙폭(MDD) / 샤프 비율
승률 / 평균 손익비(Profit Factor) / 총 거래 횟수
평균 보유 기간
```

### 차트

- Equity Curve vs 코스피 벤치마크
- 매매 시점 마커 (캔들차트 위 BUY/SELL)
- 지표 오버레이 (MA, RSI, MACD 서브차트)
- 월별 수익률 히트맵
- Drawdown 차트

### 사후 비교 분석 (Counterfactual)

- 실제 운용 기간 동안 다른 전략을 썼다면? → 자동 비교 백테스팅
- "MACD 전략을 썼다면 +8.3% 더 수익" 형태로 표시

---

## 11. 알림 / 모니터링 (웹 UI에서 ON/OFF 설정)

### Slack 알림 종류

| 알림 | 트리거 | 기본값 |
|---|---|---|
| 매매 신호 발생 | T일 종가 기준 신호 생성 시 | ON |
| 주문 체결 | KIS 체결 확인 시 | ON |
| 일일 리포트 | 매일 16:30 (장 마감 후) | ON |
| 이상 감지 알림 | 급락/서킷브레이커 | ON |
| 로그인 실패/계정 잠금 | 보안 이벤트 | ON (변경 불가) |
| 주간 성과 요약 | 매주 금요일 17:00 | OFF |

### 이상 감지 자동 처리

```
개별 종목 급락       임계값(default: -5%) 이하 시 자동 손절 (default: OFF)
포트폴리오 급락      전체 자산 일중 -X% 시 전 종목 청산 (default: OFF)
서킷브레이커 발동    KRX 서킷브레이커 → 신규 매수 자동 중단
거래 정지 종목       감지 시 알림 + 해당 종목 주문 차단
```

### 일일 리포트 항목

```
당일 실현 손익
현재 보유 종목 + 평가손익
활성 전략 상태 (정상/경고)
시장 상황 요약 (코스피 등락률)
다음 거래일 예정 매매 신호 미리보기
```

---

## 12. 데이터베이스 설계

### 시장 데이터

```sql
stocks (
  ticker        VARCHAR(10) PK,
  name          VARCHAR(100),
  market        VARCHAR(10),       -- KOSPI | KOSDAQ
  sector        VARCHAR(50),
  market_cap    BIGINT,
  listed_date   DATE,
  delisted      BOOLEAN DEFAULT FALSE,
  updated_at    TIMESTAMP
)

price_daily (
  ticker        VARCHAR(10) FK,
  date          DATE,
  open          NUMERIC(12,2),
  high          NUMERIC(12,2),
  low           NUMERIC(12,2),
  close         NUMERIC(12,2),
  volume        BIGINT,
  adj_close     NUMERIC(12,2),     -- 수정주가
  PRIMARY KEY (ticker, date)
)

price_minute (
  ticker        VARCHAR(10) FK,
  datetime      TIMESTAMP,
  open, high, low, close  NUMERIC(12,2),
  volume        BIGINT,
  PRIMARY KEY (ticker, datetime)
)

stock_fundamentals (
  ticker        VARCHAR(10) FK,
  date          DATE,              -- 기준일 (분기/연간)
  per           NUMERIC(8,2),
  pbr           NUMERIC(8,2),
  roe           NUMERIC(8,2),
  eps           NUMERIC(12,2),
  PRIMARY KEY (ticker, date)
)
```

### 사용자 / 계좌

```sql
users (
  id            BIGSERIAL PK,
  email         VARCHAR(255) UNIQUE,
  password_hash VARCHAR(255),
  name          VARCHAR(100),
  role          VARCHAR(10) DEFAULT 'USER',  -- USER | ADMIN
  totp_secret   VARCHAR(255),               -- TOTP 시크릿 (암호화)
  totp_enabled  BOOLEAN DEFAULT FALSE,
  allowed_ips   JSONB,                      -- ADMIN IP 화이트리스트
  is_locked     BOOLEAN DEFAULT FALSE,
  login_fail_count INTEGER DEFAULT 0,
  slack_webhook_url VARCHAR(500),
  notification_settings JSONB,
  created_at    TIMESTAMP
)

refresh_tokens (
  id            BIGSERIAL PK,
  user_id       BIGINT FK,
  token_hash    VARCHAR(255),
  expires_at    TIMESTAMP,
  revoked       BOOLEAN DEFAULT FALSE,
  created_at    TIMESTAMP
)

accounts (
  id            BIGSERIAL PK,
  user_id       BIGINT FK,
  type          VARCHAR(10),        -- REAL | SIM
  name          VARCHAR(100),
  kis_account_no   VARCHAR(20),     -- REAL만
  kis_app_key_enc  VARCHAR(500),    -- Fernet 암호화
  kis_app_secret_enc VARCHAR(500),  -- Fernet 암호화
  balance       NUMERIC(15,2) DEFAULT 0,
  is_active     BOOLEAN DEFAULT TRUE
)

positions (
  id            BIGSERIAL PK,
  account_id    BIGINT FK,
  ticker        VARCHAR(10),
  qty           INTEGER,
  avg_price     NUMERIC(12,2),
  UNIQUE (account_id, ticker)
)

orders (
  id            BIGSERIAL PK,
  account_id    BIGINT FK,
  strategy_run_id BIGINT,
  ticker        VARCHAR(10),
  side          VARCHAR(4),         -- BUY | SELL
  qty           INTEGER,
  price         NUMERIC(12,2),
  status        VARCHAR(20),        -- PENDING | FILLED | CANCELLED | FAILED
  signal_date   DATE,               -- 신호 발생일 (T)
  executed_at   TIMESTAMP           -- 실행일 (T+1)
)

account_daily (
  account_id    BIGINT FK,
  date          DATE,
  total_value   NUMERIC(15,2),
  cash          NUMERIC(15,2),
  holdings      JSONB,
  PRIMARY KEY (account_id, date)
)

audit_logs (
  id            BIGSERIAL PK,
  user_id       BIGINT FK,
  action        VARCHAR(50),        -- ORDER_PLACED | STRATEGY_ACTIVATED | ...
  target_type   VARCHAR(50),
  target_id     BIGINT,
  detail        JSONB,
  ip_address    VARCHAR(45),
  created_at    TIMESTAMP
)
```

### 전략 / 백테스팅

```sql
strategies (
  id            BIGSERIAL PK,
  user_id       BIGINT FK,
  name          VARCHAR(100),
  algorithm_type VARCHAR(20),
  params        JSONB,              -- 알고리즘 파라미터
  trade_params  JSONB,              -- 공통 매매 조건
  description   TEXT,
  is_public     BOOLEAN DEFAULT FALSE,
  created_at    TIMESTAMP
)

backtest_runs (
  id            BIGSERIAL PK,
  strategy_id   BIGINT FK,
  user_id       BIGINT FK,
  name          VARCHAR(100),
  tickers       JSONB,
  start_date    DATE,
  end_date      DATE,
  initial_capital NUMERIC(15,2),
  validation_type VARCHAR(20),      -- SIMPLE | WALK_FORWARD | OPTIMIZE
  validation_params JSONB,
  status        VARCHAR(20),        -- PENDING | RUNNING | DONE | FAILED
  error_message TEXT,
  created_at    TIMESTAMP,
  completed_at  TIMESTAMP
)

backtest_metrics (
  run_id              BIGINT FK PK,
  total_return_pct    NUMERIC(10,4),
  annualized_return   NUMERIC(10,4),
  benchmark_return    NUMERIC(10,4), -- 코스피 동기간 수익률
  alpha               NUMERIC(10,4),
  mdd_pct             NUMERIC(10,4),
  sharpe_ratio        NUMERIC(10,4),
  win_rate            NUMERIC(10,4),
  profit_factor       NUMERIC(10,4),
  total_trades        INTEGER,
  avg_holding_days    NUMERIC(8,2)
)

backtest_trades (
  id            BIGSERIAL PK,
  run_id        BIGINT FK,
  ticker        VARCHAR(10),
  date          DATE,
  action        VARCHAR(4),
  price         NUMERIC(12,2),
  qty           INTEGER,
  pnl           NUMERIC(12,2),
  balance_after NUMERIC(15,2)
)

backtest_equity_curve (
  run_id          BIGINT FK,
  date            DATE,
  portfolio_value NUMERIC(15,2),
  benchmark_value NUMERIC(15,2),
  PRIMARY KEY (run_id, date)
)

strategy_activations (
  id              BIGSERIAL PK,
  account_id      BIGINT FK,
  strategy_id     BIGINT FK,
  backtest_run_id BIGINT FK,         -- 근거가 된 백테스트
  started_at      TIMESTAMP,
  stopped_at      TIMESTAMP,
  stop_reason     VARCHAR(50)        -- USER | ANOMALY | LOSS_LIMIT | ADMIN
)
```

---

## 13. API 설계 (Backend :8000)

```
# 인증
POST /auth/register
POST /auth/login                    → { access_token, refresh_token }
POST /auth/refresh
POST /auth/logout
GET  /auth/me
POST /auth/totp/setup               → TOTP QR 코드 발급
POST /auth/totp/verify              → TOTP 활성화 확인

# 시장 데이터
GET  /market/stocks                 → 종목 목록 (섹터/시가총액/거래량/PER 필터)
GET  /market/stocks/{ticker}/price  → OHLCV (timeframe, from, to)
GET  /market/stocks/{ticker}/indicators
GET  /market/sectors                → 섹터 목록 + 강세/약세 현황

# 전략
GET    /strategies
POST   /strategies
GET    /strategies/{id}
PUT    /strategies/{id}
DELETE /strategies/{id}

# 백테스팅
POST /backtest/run
GET  /backtest/runs
GET  /backtest/runs/{id}            → metrics + trades + equity curve
POST /backtest/compare              → N개 전략 동시 비교
POST /backtest/counterfactual       → 실제 운용 기간 사후 비교
WS   /ws/backtest/{run_id}          → 진행률 스트리밍

# 계좌
GET  /accounts
POST /accounts
GET  /accounts/{id}/positions
GET  /accounts/{id}/orders
GET  /accounts/{id}/history

# 자동매매 (시뮬 전용, :8000)
POST /trading/activate              → 시뮬 전략 활성화
POST /trading/deactivate
GET  /trading/active
WS   /ws/trading                    → 실시간 신호/체결

# 관리 (ADMIN 전용, :8000)
GET  /admin/users
PUT  /admin/users/{id}/lock
GET  /admin/audit-logs

# 설정
GET  /settings/notifications
PUT  /settings/notifications
POST /settings/slack/test

# ── Real Trading Service (:8002, 내부망 only) ──────────────
POST /real/trading/activate         → 실계좌 전략 활성화
POST /real/trading/deactivate
POST /real/order                    → 수동 주문
GET  /real/balance                  → 실계좌 잔고
GET  /real/positions                → 보유 포지션
GET  /real/orders                   → 체결 내역
WS   /real/ws                       → KIS WebSocket 실시간 체결
```

---

## 14. Backtest Worker API (Mac Mini :8001)

```
POST /backtest
  Request:
    algorithm_type, params, trade_params,
    tickers, prices (OHLCV),
    start_date, end_date, initial_capital,
    validation_type, validation_params

  Response:
    metrics { total_return, mdd, sharpe, alpha, win_rate, ... }
    trades  [{ date, ticker, action, price, qty, pnl }]
    equity_curve [{ date, portfolio_value, benchmark_value }]

GET /health
```

---

## 15. 프론트엔드 화면

| 화면 | 경로 | 주요 기능 |
|---|---|---|
| 대시보드 | `/` | 포트폴리오 요약, 활성 전략, 최근 매매, 수익률 차트 |
| 종목 탐색 | `/market` | 섹터/조건 스크리닝, 차트, 지표 오버레이 |
| 전략 관리 | `/strategies` | 전략 목록, 생성/수정 |
| 전략 상세 | `/strategies/:id` | 파라미터 설정 폼 (추천값 표시) |
| 백테스팅 | `/backtest/new` | 전략 선택, 종목 스크리닝, 기간/검증 옵션 |
| 백테스트 결과 | `/backtest/:id` | Equity curve, 매매 마커, 지표 차트, metrics |
| 전략 비교 | `/backtest/compare` | N개 전략 동시 비교 차트 |
| 자동매매 (시뮬) | `/trading` | 시뮬 전략 활성화/관리, 이상 감지 설정 |
| 포트폴리오 | `/portfolio/:accountId` | 보유종목, 주문이력, 자산추이 |
| 설정 | `/settings` | Slack 연동, 알림 설정, IP 화이트리스트 |
| 관리자 | `/admin` | 유저 관리, 감사 로그 (ADMIN만) |
| **실전 매매** | `localhost:8002` (내부망 전용) | 실계좌 전략 활성화, 수동 주문, 잔고/체결 |

---

## 16. MVP 개발 순서

### Phase 1 — 인프라 + 인증 기반 (1주)
- [ ] Docker Compose 기본 구성 (postgres, redis, backend, frontend)
- [ ] PostgreSQL 스키마 + Alembic 마이그레이션
- [ ] JWT 인증 (Access 15분 + Refresh 7일)
- [ ] RBAC (USER / ADMIN)
- [ ] 감사 로그 미들웨어
- [ ] 로그인 실패 잠금 + Slack 보안 알림

### Phase 2 — 데이터 파이프라인 (1-2주)
- [ ] pykrx: KOSPI 전종목 리스트 수집
- [ ] pykrx: 일봉 3년치 벌크 적재 (수정주가 포함)
- [ ] pykrx: 재무 데이터 (PER, PBR, ROE)
- [ ] APScheduler: 일별 cron 업데이트
- [ ] KIS 토큰 자동 갱신 데몬 (24h 만료 대응)

### Phase 2-b — (선택) TimescaleDB 전환
- [ ] PostgreSQL TimescaleDB 확장 설치
- [ ] price_daily, price_minute → hypertable 전환
- [ ] 쿼리 성능 비교 후 적용 여부 결정

### Phase 3 — 백테스팅 코어 (2-3주)
- [ ] Redis + Celery Worker 구성 (Mac Mini)
- [ ] Backtest Worker: MA_CROSS, RSI 구현 (pandas-ta)
- [ ] Backtest Worker: metrics + 벤치마크(코스피) 비교
- [ ] Backtest Worker: equity curve 생성
- [ ] API: /strategies, /backtest CRUD + Celery 태스크 디스패치
- [ ] Frontend: 전략 파라미터 UI (추천값 표시)
- [ ] Frontend: TradingView 차트 + equity curve + 매매 마커
- [ ] Frontend: 종목 스크리닝 + 섹터 필터
- [ ] WebSocket 백테스트 진행률 스트리밍

### Phase 4 — 전략 확장 + 비교 분석 (2주)
- [ ] MACD, BOLLINGER, MOMENTUM, STOCHASTIC 추가
- [ ] MULTI 조건 조합
- [ ] Walk-Forward 검증
- [ ] 전략 비교 + Counterfactual 사후 비교
- [ ] 파라미터 최적화 (그리드 서치)

### Phase 5 — 시뮬레이션 매매 (2주)
- [ ] KIS API 인증 모듈 (가상계좌)
- [ ] T+1 신호 기반 자동 주문
- [ ] 포트폴리오 대시보드
- [ ] Slack 매매 알림 + 일일 리포트
- [ ] 이상 감지 + 자동 처리
- [ ] WebSocket 실시간 알림

### Phase 6 — 실전 매매 (2주, 내부망 전용)
- [ ] Real Trading Service (:8002) 신규 FastAPI 앱 구성
- [ ] KIS 실계좌 REST 연동 (잔고, 주문, 체결)
- [ ] KIS WebSocket 실시간 체결 알림
- [ ] 실전 전략 활성화 + T+1 주문 실행
- [ ] 리스크 관리 (일일 손실 한도, 포지션 사이징)
- [ ] 전략 활성화 이력 + 감사 로그
- [ ] Docker Compose: :8002 외부 포트 미노출 + Nginx `allow 192.168.x.x; deny all;` 이중 차단

### Phase 7 — 커스텀 코드 샌드박스 (1주)
- [ ] Mac Mini Docker daemon 설정
- [ ] 에피머럴 컨테이너 실행 로직 (`--network none --memory 512m --cpus 1`)
- [ ] 실행 결과 수집 + 컨테이너 자동 삭제
- [ ] 타임아웃 + 비정상 종료 처리

---

## 17. 인프라

```yaml
# docker-compose.yml (Windows)
services:
  frontend:        next.js        ports: "3000:3000"    ← Cloudflare 노출
  backend:         fastapi        ports: "8000:8000"    ← Cloudflare 노출
  redis:           redis:7        ports 미설정           ← 내부망 only
  real-trading:    fastapi        ports 미설정           ← 내부망 only (localhost:8002)
  postgres:        postgres:17    ports 미설정           ← 내부망 only
  collector:       python + APScheduler (pykrx)
  kis-refresher:   python daemon  ← KIS 토큰 자동 갱신
  cloudflared:     cloudflare tunnel

# Mac Mini (Docker)
  celery-worker:   python + celery  ← Redis 큐 소비
  docker daemon:   커스텀 코드 샌드박스용
```

---

## 18. 기존 kis-trader 구조 비교 분석

### 재사용 가능 (그대로 or 확장)

| 기존 파일/모듈 | 재사용 방향 |
|---|---|
| `src/backtest_engine/main.py` | RSI 구현 + metrics 계산 → 새 알고리즘 추가하며 확장 |
| `src/live_engine/main.py` | KIS 주문/시세/잔고 → Real Trading Service(:8002)로 이전 |
| `src/kis_api.py` | KIS API 래퍼 → 그대로 재사용 |
| `src/strategies/base_strategy.py` | 전략 기본 클래스 → 새 알고리즘 구현 베이스로 활용 |
| `src/strategies/rsi_strategy.py` | RSI 전략 로직 참조 |
| `cloudflare/config.yml` | 서브도메인만 수정해서 재사용 |
| DB 파티셔닝 구조 (`daily_prices`, `minute_prices`) | 스키마 확장하며 유지 |
| `docker-compose.yml` 구조 | 서비스 교체하며 재활용 |

### 전면 교체

| 기존 | 교체 | 이유 |
|---|---|---|
| **Spring Boot 3.4 (Java)** Backend | **FastAPI (Python)** | 단일 언어 스택, 연산 레이어 통일 |
| Spring Security + STOMP WebSocket | FastAPI JWT + native WebSocket | Spring 의존성 제거 |
| Spring Batch (데이터 수집) | pykrx + APScheduler | Python 생태계 통일, KRX 직접 연동 |
| OpenFeign (HTTP 직접 호출) | Redis + Celery 태스크 큐 | 장시간 백테스트 안정성 |
| React Flow 노드 전략 빌더 | 파라미터 폼 + 프리셋 방식 | 퀀트 알고리즘은 노드보다 폼이 직관적 |
| recharts | TradingView Lightweight Charts | 캔들차트 + 매매 마커 필수 |
| `@stomp/stompjs` | FastAPI WebSocket 직접 | STOMP 불필요 |

### 신규 추가 (기존에 없음)

| 항목 | 비고 |
|---|---|
| Redis + Celery | Mac Mini 태스크 큐 |
| pykrx 데이터 파이프라인 | KOSPI 전종목 3년치 수집 |
| KIS 토큰 자동 갱신 데몬 | 24h 만료 대응 |
| Slack 알림 (매매신호, 일일리포트, 이상감지) | |
| Walk-Forward / 파라미터 최적화 | 백테스팅 품질 강화 |
| 전략 비교 + Counterfactual 분석 | |
| 종목 스크리닝 (섹터, PER, PBR, ROE) | |
| Real Trading Service 분리 (:8002, 내부망) | 실계좌 완전 격리 |
| Docker 에피머럴 컨테이너 샌드박스 | 커스텀 코드 실행 |
| 감사 로그 (audit_logs) | 실전 매매 보안 |
| TimescaleDB 옵션 | 시계열 성능 |
| TOTP 2차 인증 | ADMIN 보호 |
| my-ui-lib 적용 | 공용 UI 컴포넌트 |

### 마이그레이션 전략

```
1. 신규 브랜치에서 FastAPI 백엔드 스캐폴딩 (기존 Spring Boot와 병행 운영)
2. DB 스키마: Flyway 마이그레이션 → Alembic으로 전환
   - 기존 테이블 구조 최대한 호환, 신규 컬럼/테이블만 추가
3. 기존 backtest_engine, live_engine Python 코드는 그대로 이식
4. 프론트엔드: 기존 Next.js 앱 구조 유지, UI 컴포넌트를 my-ui-lib으로 교체
5. React Flow 전략 빌더는 Phase 3 이후 필요 시 재도입 검토
```

---

### 환경변수 (.env, 절대 커밋 금지)

```
DATABASE_URL
REDIS_URL
KIS_ENCRYPT_KEY          # Fernet 키
JWT_SECRET_KEY
SLACK_BOT_TOKEN
BACKTEST_WORKER_URL      # Mac Mini LAN (레거시, Celery 전환 후 제거)
```

### Cloudflare Tunnel 라우팅

```
app.도메인.com → frontend :3000
api.도메인.com → backend  :8000
```

### 환경변수 (`.env`, 절대 커밋 금지)

```
DATABASE_URL
KIS_ENCRYPT_KEY         # Fernet 키 (KIS API 암호화용)
JWT_SECRET_KEY
SLACK_BOT_TOKEN
BACKTEST_WORKER_URL     # Mac Mini LAN 주소
```
