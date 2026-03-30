# Infrastructure

> 홈 서버 기반 자체 운영 인프라. 모든 외부 노출은 Cloudflare Tunnel 경유.

---

## 전체 네트워크 구조

```mermaid
graph LR
    User([🌐 User])
    GHA[⚙️ GitHub Actions]

    subgraph CF["☁️ Cloudflare"]
        DNS[DNS]
        TW[Tunnel\nWindows]
        TM[Tunnel\nMac Mini]
    end

    subgraph Vercel["▲ Vercel"]
        BG[boldgobynd]
    end

    User --> DNS
    DNS --> TW & TM & Vercel
    GHA -->|SSH :2222| TW
    GHA -->|SSH :22| TM
```

---

## Windows Server — 서비스

```mermaid
graph TB
    TW[Cloudflare Tunnel]

    subgraph PM2["📦 PM2"]
        profile["profile\nprofile.nuclearbomb6518.com"]
        storybook["storybook\nstorybook.nuclearbomb6518.com"]
        seobi["seobi-chat\nseobi.nuclearbomb6518.com"]
    end

    subgraph TechFeed["🐳 Docker — TechFeed"]
        tf_api["API (NestJS)\ntechfeed-api.nuclearbomb6518.com"]
        tf_crawler[Crawler]
        tf_mongo[(MongoDB)]
        tf_redis[(Redis)]
        tf_es[(Elasticsearch)]
        tf_pg[(PostgreSQL\nTimescaleDB :3104)]
    end

    subgraph KisTrader["🐳 Docker — KIS Trader"]
        kt_front["Frontend (Next.js)\ntrader.nuclearbomb6518.com"]
        kt_back["Backend (FastAPI)\ntrader-api.nuclearbomb6518.com"]
        kt_workers[data-collector\nreal-trading\nbacktest-worker]
        kt_redis[(Redis)]
    end

    lotto["🐳 lotto-oracle\nlotto.nuclearbomb6518.com"]

    TW --> profile & storybook & seobi
    TW --> tf_api & kt_front & kt_back & lotto

    tf_api --- tf_mongo & tf_redis & tf_es & tf_pg
    tf_crawler --- tf_mongo & tf_redis

    kt_front --> kt_back
    kt_back --- tf_pg & kt_redis
    kt_workers --- tf_pg & kt_redis
```

---

## Mac Mini — DevOps Monitor

```mermaid
graph TB
    TM[Cloudflare Tunnel]

    subgraph Monitor["🐳 Docker — devops-monitor"]
        grafana["Grafana\nmonitoring.nuclearbomb6518.com"]
        prometheus[Prometheus]
        loki[Loki]
        alertmanager[Alertmanager → Slack]
        timescaledb[(TimescaleDB\nanalytics DB)]
        ingestor["Ingestor\ningestor.nuclearbomb6518.com"]
        blackbox[Blackbox Exporter\n외부 URL 헬스체크]
    end

    subgraph AgentsWin["📡 Windows Agents"]
        node_win[node-exporter]
        cadvisor_win[cAdvisor]
        loki_win[Loki]
        promtail[promtail]
    end

    TM --> grafana & ingestor

    grafana --> prometheus & loki & timescaledb
    prometheus --> node_win & cadvisor_win & blackbox
    loki --> loki_win
    promtail --> loki_win
    alertmanager -.- prometheus
    ingestor --> timescaledb
```

---

상세 정보 → [services.md](./services.md)
