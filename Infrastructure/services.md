# Services — 호스트별 배포/운영 현황

> 마지막 확인: 2026-03-29
> 모든 외부 노출은 Cloudflare Tunnel 경유 (포트 직접 오픈 없음)

---

## Windows Server (배포 PC)

| 항목 | 내용 |
|------|------|
| 호스트명 | → `secrets/credentials.md` |
| 내부 IP | → `secrets/credentials.md` |
| 외부 접근 | DDNS — `windows.nuclearbomb6518.com` |
| OS | Windows 10 |
| 프로세스 관리 | PM2 + Windows Task Scheduler (`PM2-Resurrect`) |
| 컨테이너 | Docker Desktop |
| 터널 | Cloudflare Tunnel (UUID → `secrets/credentials.md`) |

### 포트별 서비스

| 포트 | 서비스 | 스택 | 외부 도메인 | 프로세스 |
|------|--------|------|-------------|----------|
| `22` (외부 `2222`) | SSH | OpenSSH | `windows.nuclearbomb6518.com:2222` | — |
| `3000` | **profile** | Next.js standalone | `profile.nuclearbomb6518.com` | PM2 (`profile-next`) |
| `3001` | **lotto-oracle** | FastAPI + Docker | `lotto.nuclearbomb6518.com` | Docker (`lotto-oracle-lotto-oracle-1`) |
| `3002` | **seobi-chat** | Node.js | `seobi.nuclearbomb6518.com` | PM2 (`seobi-chat`) |
| `3100` | **techfeed-api** | NestJS + Docker | `techfeed-api.nuclearbomb6518.com` | Docker (`techfeed-api`) |
| `3200` | **kis-trader-frontend** | Next.js + Docker | `trader.nuclearbomb6518.com` | Docker (`kis-trader-frontend-1`) |
| `6006` | **storybook** | Storybook (my-ui-lib) | `storybook.nuclearbomb6518.com` | PM2 (`storybook`) |
| `8000` | **kis-trader-backend** | FastAPI + Docker | `trader-api.nuclearbomb6518.com` | Docker (`kis-trader-backend-1`) |

### kis-trader Docker 서비스

| 컨테이너 | 역할 | 포트 |
|---------|------|------|
| `kis-trader-backend-1` | FastAPI 백엔드 API | `8000` (host) |
| `kis-trader-frontend-1` | Next.js 프론트엔드 | `3200` (host) |
| `kis-trader-data-collector-1` | 주식 데이터 수집 스케줄러 | — |
| `kis-trader-real-trading-1` | 실전 매매 서비스 | — |
| `kis-trader-backtest-worker-1` | 백테스트 Celery 워커 | — |
| `kis-trader-redis-1` | Redis (Celery broker/backend) | 내부 only |

> DB: techfeed-postgres(`3104`)의 `kistrader` 데이터베이스 공유 사용
> CI/CD: GitHub Actions push to main → SSH → docker compose up
> 배포 URL: https://trader.nuclearbomb6518.com
> Admin 계정: → `secrets/credentials.md`
> 분석 이벤트: pageview, login, backtest_run, simulation_activate → ingestor.nuclearbomb6518.com

### Cloudflare Tunnel ingress

```yaml
- windows.nuclearbomb6518.com      → ssh://localhost:22
- profile.nuclearbomb6518.com      → http://localhost:3000
- lotto.nuclearbomb6518.com        → http://localhost:3001
- seobi.nuclearbomb6518.com        → http://localhost:3002
- storybook.nuclearbomb6518.com    → http://localhost:6006
- techfeed-api.nuclearbomb6518.com → http://localhost:3100
- trader.nuclearbomb6518.com       → http://localhost:3200
- trader-api.nuclearbomb6518.com   → http://localhost:8000
```

### techfeed DB 컨테이너 (Docker)

> SSH 터널(`db-open`)로 Mac에서 접근 가능

| 포트 | 컨테이너 | 스택 | 용도 | DB 접속 정보 |
|------|---------|------|------|-------------|
| `3101` | `techfeed-mongodb` | MongoDB 7 | 콘텐츠 (블로그/유튜브/채용공고) | `mongodb://localhost:3101/techfeed` |
| `3102` | `techfeed-elasticsearch` | Elasticsearch 9 | 콘텐츠 검색 인덱스 | `http://localhost:3102` |
| `3103` | `techfeed-redis` | Redis 7 | 랭킹 캐시, BullMQ 큐 | `localhost:3103` |
| `3104` | `techfeed-postgres` | TimescaleDB (pg15) | users, bookmarks, subscriptions, events (techfeed) / kis-trader data (kistrader DB) | `postgresql://techfeed:<password>@localhost:3104/techfeed` (→ `secrets/credentials.md`) |

### devops-monitor 에이전트 (Windows)

Windows 에이전트는 `docker-compose.windows-agents.yml`로 관리.
GitHub Actions push to main → SSH → `docker compose -f docker-compose.windows-agents.yml up -d`

| 포트 | 컨테이너 | 역할 |
|------|---------|------|
| `9100` | node-exporter | Windows OS 메트릭 |
| `8080` | cadvisor | Docker 컨테이너 메트릭 |
| `9300` | metrics-proxy (nginx) | 메트릭 프록시 (PM2 메트릭, standalone) |
| — | loki | 로그 수신 (Windows 로컬, promtail push 대상) |
| — | promtail | Docker 컨테이너 로그 수집 → loki push |
| — | pm2-prometheus-exporter | PM2 프로세스 메트릭 (PM2, standalone) |

### PM2 프로세스 현황

| id | name | 상태 |
|----|------|------|
| 0 | pm2-prometheus-exporter | online |
| 1 | profile-next | online |
| 2 | storybook | online |
| 6 | seobi-chat | online |

---

## Mac Mini (로컬)

| 항목 | 내용 |
|------|------|
| 역할 | 개발 머신 + 모니터링 서버 (Hub) |
| 프로세스 관리 | Docker Compose |
| 터널 | Cloudflare Tunnel (UUID → `secrets/credentials.md`, homebrew 서비스) |

### devops-monitor 스택

> 모든 포트는 `127.0.0.1` 바인딩. 외부 접근은 Cloudflare Tunnel 경유.

| 포트 | 컨테이너 | 역할 | 외부 도메인 |
|------|---------|------|------------|
| `3000` | grafana | 대시보드 UI | `monitoring.nuclearbomb6518.com` |
| `3100` | loki | 로그 수집 | 내부 only |
| `4000` | ingestor | 유저 이벤트 수집 API | `ingestor.nuclearbomb6518.com` |
| `5432` | timescaledb | 유저 이벤트 시계열 DB (`analytics` / `monitor`) | 내부 only |
| `8080` | cadvisor | Docker 컨테이너 메트릭 | 내부 only |
| `9090` | prometheus | 메트릭 수집 (15일 보존) | 내부 only |
| `9093` | alertmanager | 알림 라우팅 → Slack | 내부 only |
| `9100` | node-exporter | Mac Mini OS 메트릭 | 내부 only |
| `9115` | blackbox-exporter | 외부 URL HTTP 프로브 | 내부 only |

> 실행: `make up` / 레포: `Projects/devops-monitor`

### Cloudflare Tunnel ingress (Mac)

```yaml
tunnel: <mac-tunnel-uuid>  # → secrets/credentials.md
ingress:
  - monitoring.nuclearbomb6518.com → http://localhost:3000
  - ingestor.nuclearbomb6518.com   → http://localhost:4000
  - mac.nuclearbomb6518.com        → ssh://localhost:22
```

---

## Vercel

| 서비스 | 도메인 | 레포 |
|--------|--------|------|
| **boldgobynd** (studiobold) | `boldgobynd.vercel.app` | `Junseop-Shin/boldgobynd` |

> Vercel 환경변수: `NEXT_PUBLIC_INGESTOR_URL=https://ingestor.nuclearbomb6518.com`
> Blackbox Exporter로 UP/DOWN 모니터링 중 (`probe_success{service="studiobold"}`)

---

## DB 접속 (개발 환경)

> Mac에서 `db-open` 명령 실행 후 아래 정보로 연결

```bash
db-open    # PostgreSQL:5432 + Redis:6379 + ES:9200 SSH 터널 오픈
db-close   # 터널 닫기
db-status  # 터널 상태 확인
```

| DB | 툴 | Host | Port | DB명 | User | Password |
|----|-----|------|------|------|------|----------|
| PostgreSQL (techfeed) | DBeaver | localhost | 5432 | techfeed | techfeed | → `secrets/credentials.md` |
| PostgreSQL (kistrader) | DBeaver | localhost | 5432 | kistrader | kistrader | GitHub Secret: `POSTGRES_PASSWORD` |
| TimescaleDB (모니터링) | DBeaver | localhost | 5432 | analytics | monitor | → `secrets/credentials.md` |
| MongoDB (techfeed) | Compass | localhost | 3101 | techfeed | — | — |
| Redis (techfeed) | RedisInsight | localhost | 6379 | — | — | — |
| Elasticsearch (techfeed) | Elasticvue | localhost | 9200 | — | — | — |

> MongoDB는 Compass 내 SSH 터널로 별도 연결 (`windows.nuclearbomb6518.com:2222`)

---

## 배포 방식 요약

| 서비스 | CI/CD | 배포 방법 |
|--------|-------|-----------|
| techfeed-api | GitHub Actions (push to main) | SSH → docker compose up --build |
| techfeed-crawler | GitHub Actions (push to main) | SSH → docker compose up --build |
| profile | GitHub Actions (push to main) | SSH → tar 전송 → pm2 reload |
| lotto-oracle | GitHub Actions (push to main) | SSH → docker compose up |
| seobi-chat | GitHub Actions (push to main) | SSH → tar 전송 → pm2 reload |
| storybook (my-ui-lib) | GitHub Actions (push to main) | SSH → tar 전송 → pm2 serve |
| boldgobynd | Vercel (push to main 자동) | — |
| devops-monitor | GitHub Actions (push to main) | SSH → Mac Mini → docker compose up --build |
| kis-trader | GitHub Actions (push to main) | SSH → tar 전송 → docker compose up --build |

---

## 미배포 프로젝트

| 프로젝트 | 상태 |
|---------|------|
| techfeed (mobile) | EAS 빌드 (internal APK 배포), 기능 개발 진행 중 |
| my-ui-lib | npm 패키지 배포 / storybook으로 UI 확인 |
