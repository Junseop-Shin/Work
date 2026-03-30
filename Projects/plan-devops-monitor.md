# Project Plan: devops-monitor

> 자체 배포 서비스 전체를 모니터링하는 옵저버빌리티 플랫폼
> Status: Planning
> Created: 2026-03-23

---

## 목표

- 맥미니(로컬) + 배포 PC에서 운영 중인 모든 서비스를 단일 대시보드에서 모니터링
- 시스템 메트릭(CPU/메모리/디스크) + 서비스 상태 + 로그 + 유저 이벤트 통합
- 이상 감지 시 Slack 알림 자동 발송

---

## 아키텍처 — Hub & Spoke

```
[배포 PC]  ──── Tailscale VPN ────  [맥미니 (Hub)]
  node_exporter                       Prometheus (수집)
  cAdvisor                            Loki (로그)
  Promtail                            TimescaleDB (유저 이벤트)
  서비스 /metrics                     Grafana (시각화)
                                      AlertManager → Slack
                                      Event Ingestor API
                                      Nginx (리버스 프록시)
```

맥미니와 배포 PC 간 통신은 **Cloudflare Tunnel**로 보안 처리.
각 호스트에 `cloudflared` 데몬만 설치하면 포트 오픈 없이 HTTPS 터널 구성.
Prometheus/Loki 포트는 외부 인터넷에 직접 노출하지 않음.

---

## 인프라 구성

### 모니터링 서버 — 맥미니 (Docker Compose)

| 컨테이너 | 역할 | 포트 |
|---------|------|------|
| Prometheus | 메트릭 수집 및 저장 | 9090 |
| Grafana | 시각화 대시보드 | 3000 |
| Loki | 로그 집계 | 3100 |
| Promtail | 로컬 로그 수집 → Loki | - |
| AlertManager | 알림 라우팅 → Slack | 9093 |
| TimescaleDB | 유저 이벤트 시계열 저장 | 5432 |
| Event Ingestor | 이벤트 수집 API (Node.js) | 4000 |
| Nginx | 리버스 프록시 + Basic Auth | 80/443 |

### 각 호스트에 설치할 에이전트

| 에이전트 | 역할 | 포트 |
|---------|------|------|
| node_exporter | OS 메트릭 (CPU, 메모리, 디스크, 네트워크) | 9100 |
| cAdvisor | Docker 컨테이너별 리소스 | 8080 |
| Promtail | 로그 수집 → Loki 전송 | - |

PM2로 운영하는 서비스는 `/metrics` 엔드포인트 추가 필요.

---

## 기술 스택

| 카테고리 | 기술 | 선택 이유 |
|---------|------|---------|
| 메트릭 수집 | Prometheus | Pull 기반, VPN 환경에서 방화벽 설정 단순 |
| 메트릭 에이전트 | node_exporter, cAdvisor | OS/컨테이너 메트릭 업계 표준 |
| 로그 수집 | Loki + Promtail | Prometheus와 동일한 레이블 체계, Grafana 통합 |
| 시각화 | Grafana | 메트릭 + 로그 + SQL 단일 대시보드 |
| 알림 | AlertManager + Slack Webhook | 알림 그룹핑, 라우팅 룰 지원 |
| 이벤트 DB | TimescaleDB | PostgreSQL 기반 + Hypertable로 시계열 쿼리 최적화 |
| 네트워크 | Cloudflare Tunnel | cloudflared 데몬만으로 포트 오픈 없이 보안 연결, HTTPS 자동 |
| 인프라 | Docker Compose | 전체 스택 단일 파일 관리, 백업 용이 |
| 리버스 프록시 | Nginx | 포트 통합 노출 + Basic Auth |

---

## 데이터 모델

### TimescaleDB — 유저 이벤트 Hypertable

```sql
CREATE TABLE user_events (
    time        TIMESTAMPTZ NOT NULL,
    user_id     UUID,
    event_type  VARCHAR(50),   -- 'page_view', 'click', 'signup' 등
    service_id  VARCHAR(50),   -- 어느 서비스에서 발생했는지
    metadata    JSONB,         -- 유입경로, UTM, 페이지명 등 유연한 속성
    ip_address  INET
);
SELECT create_hypertable('user_events', 'time');
```

---

## Event Ingestor API

앱에서 TimescaleDB에 직접 쓰지 않고 Ingestor 서비스를 경유.
배치 쓰기로 IOPS 최소화.

```
POST /v1/events
{
  "user_id": "uuid",
  "event_type": "page_view",
  "service_id": "profile",
  "metadata": {
    "referrer": "https://google.com",
    "utm_source": "twitter",
    "page": "/about"
  }
}
```

---

## 알림 정책

| 조건 | 채널 | 심각도 |
|------|------|--------|
| 서비스 다운 (5분 이상) | #alert-critical | Critical |
| 5xx 에러율 5% 초과 | #alert-warning | Warning |
| CPU 80% 초과 (10분 지속) | #alert-warning | Warning |
| 메모리 90% 초과 | #alert-critical | Critical |
| 디스크 85% 초과 | #alert-warning | Warning |
| 응답시간 2초 초과 | #alert-warning | Warning |

---

## 서비스별 /metrics 연동

### Node.js / Express
```js
const promClient = require('prom-client');
promClient.collectDefaultMetrics();
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

### PM2 서비스
```bash
pm2 install pm2-metrics
# 또는 커스텀 /metrics 엔드포인트 직접 구현
```

---

## Grafana 대시보드 구성

1. **Overview** — 전체 호스트 상태 (업/다운, 리소스 요약) — "Golden Signals"
2. **Host Detail** — 특정 호스트 드릴다운 (CPU/메모리/네트워크 시계열)
3. **Service Detail** — HTTP 메트릭 (요청수, 에러율, 응답시간)
4. **Logs** — Loki 로그 검색 및 필터
5. **User Analytics** — 방문자수, 유입경로, 이벤트 (TimescaleDB SQL 패널)

---

## 구현 단계

### Phase 1 — Hub 기반 구축 (난이도: S, ~1일)
- [ ] Docker Compose: Prometheus + Grafana + Loki + AlertManager 로컬 셋업
- [ ] 맥미니에 node_exporter + cAdvisor + Promtail 설치
- [ ] Grafana 기본 대시보드 (호스트 메트릭) 구성
- [ ] Nginx 리버스 프록시 + Basic Auth 설정

### Phase 2 — 배포 PC 연동 (난이도: M, ~2일)
- [ ] 배포 PC에 node_exporter + cAdvisor + Promtail 설치
- [ ] Cloudflare Tunnel로 배포 PC 에이전트 포트 맥미니에 노출
- [ ] Prometheus scrape config에 배포 PC 타겟 추가 (Cloudflare Tunnel 경유)
- [ ] 원격 로그 수집 (Promtail → 맥미니 Loki)

### Phase 3 — 서비스 연동 (난이도: M, ~2일)
- [ ] 기존 서비스(Node.js/PM2)에 /metrics 엔드포인트 추가
- [ ] 서비스별 Grafana 대시보드 구성
- [ ] AlertManager Slack Webhook 설정 및 알림 룰 작성

### Phase 4 — 유저 이벤트 수집 (난이도: L, ~4일)
- [ ] TimescaleDB 셋업 + Hypertable 생성
- [ ] Event Ingestor Node.js 서비스 구현 (배치 쓰기)
- [ ] 기존 서비스 프론트/백엔드에 이벤트 트래킹 추가
- [ ] Grafana User Analytics 대시보드 (SQL 패널)

### Phase 5 — 고도화 (난이도: S, ~1일)
- [ ] Loki Compactor로 로그 보존 정책 설정
- [ ] Prometheus 보존 기간 설정 (`--storage.tsdb.retention.time=15d`)
- [ ] 대시보드 "Golden Signals" 완성 (Latency, Errors, Traffic, Saturation)

---

## 리스크 & 대응

| 리스크 | 대응 |
|--------|------|
| 맥미니 디스크 고갈 (로그/메트릭 누적) | Prometheus 보존기간 15일, Loki Compactor 설정 |
| 배포 PC 네트워크 보안 | Cloudflare Tunnel — 포트 직접 노출 없이 cloudflared 데몬만 운영 |
| 맥미니 장애 시 모니터링 불가 | 허용 가능한 트레이드오프 (로컬 운영의 한계) |
| Mac Mini CPU 부하 (로그 인덱싱) | Promtail 버스트 레이트 제한, Loki filesystem 스토리지 사용 |

---

## 참고

- Grafana 포트(3000)는 Nginx로만 노출, 직접 접근은 로컬만 허용
- 배포 PC 에이전트 포트는 Tailscale 네트워크 내 맥미니 IP만 허용
- Event Ingestor는 앱에서 직접 DB 접근하지 않도록 반드시 API 경유
