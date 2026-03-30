# Project Plan: TechFeed

> 개발자 뉴스 큐레이션 앱 — 검색 + 캐시 + 시계열 + 푸시 알림 통합 서비스
> Status: Planning
> Created: 2026-03-23

---

## 목표

- 기술 아티클을 수집/검색/큐레이션할 수 있는 모바일 앱 서비스
- Elasticsearch(전문 검색) + Redis(캐시/랭킹) + TimescaleDB(이벤트 분석) + FCM(푸시) 활용
- 각 기술이 "왜 쓰였는지" 명확한 포트폴리오 프로젝트

---

## 아키텍처

```
[React Native 앱]
       │
       ↓
[NestJS API Server]
   ├── 아티클 검색 → Elasticsearch
   ├── 피드 캐시 / 랭킹 → Redis
   ├── 유저 이벤트 저장 → TimescaleDB
   └── 푸시 알림 → FCM
       │
[Crawler / Scheduler]
   └── 아티클 수집 → Elasticsearch 인덱싱 → Redis Pub/Sub → FCM 트리거
```

---

## 기술 스택

| 카테고리 | 기술 | 선택 이유 |
|---------|------|---------|
| 모바일 앱 | React Native (Expo) | iOS/Android 공통, 기존 React 경험 활용 |
| 백엔드 | NestJS (TypeScript) | DI 구조, 모듈화, 대규모 확장 용이 |
| 전문 검색 | Elasticsearch | 아티클 전문 검색, 자동완성, 지리/태그 필터 |
| 캐시 / 랭킹 | Redis | 피드 캐시, 인기 키워드 Sorted Set, Pub/Sub 이벤트 |
| 이벤트 분석 | TimescaleDB | 읽기/클릭/체류 시계열 저장, 트렌드 분석 |
| 푸시 알림 | FCM (Firebase Cloud Messaging) | iOS/Android 공통, firebase-admin SDK |
| 크롤러 | Node.js + RSS/API | dev.to, Hacker News, Medium RSS 수집 |
| 스케줄러 | node-cron | 주기적 크롤링 + 예약 푸시 |
| 인프라 | Docker Compose | 로컬 개발 + 배포 일관성 |
| DB (유저/기본) | PostgreSQL | 유저, 북마크, 구독 태그 등 관계형 데이터 |

---

## 핵심 기능

### 검색 (Elasticsearch)
- 아티클 제목 + 본문 전문 검색
- 태그, 언어, 날짜 필터
- 자동완성 (Completion Suggester)
- 인기 검색어 집계

### 캐시 & 랭킹 (Redis)
- 메인 피드 캐시 (TTL 5분)
- 실시간 인기 아티클 랭킹 (Sorted Set, 조회수 기반)
- 인기 태그 랭킹
- Pub/Sub: 새 아티클 인덱싱 완료 → 구독자 알림 트리거

### 푸시 알림 (FCM)
- 관심 태그 새 아티클 등록 시 → 즉시 푸시
- 주간 트렌드 리포트 → 예약 푸시 (매주 월요일 오전)
- Redis Pub/Sub → 구독 태그 매칭 → FCM 발송 → TimescaleDB에 발송 이력 기록

### 유저 이벤트 분석 (TimescaleDB)
- 읽기, 북마크, 공유, 클릭 이벤트 시계열 저장
- 태그별 인기도 트렌드 분석
- 유저 행동 기반 개인화 추천 기반 데이터

---

## 데이터 모델

### Elasticsearch — 아티클 인덱스

```json
{
  "mappings": {
    "properties": {
      "title":       { "type": "text", "analyzer": "english" },
      "content":     { "type": "text", "analyzer": "english" },
      "tags":        { "type": "keyword" },
      "source":      { "type": "keyword" },
      "url":         { "type": "keyword" },
      "author":      { "type": "keyword" },
      "published_at":{ "type": "date" },
      "view_count":  { "type": "integer" }
    }
  }
}
```

### TimescaleDB — 유저 이벤트 Hypertable

```sql
CREATE TABLE user_events (
    time        TIMESTAMPTZ NOT NULL,
    user_id     UUID,
    event_type  VARCHAR(50),   -- 'read', 'bookmark', 'share', 'click'
    article_id  VARCHAR(100),
    tag         VARCHAR(50),
    duration_ms INTEGER,       -- 체류시간 (read 이벤트)
    metadata    JSONB
);
SELECT create_hypertable('user_events', 'time');
```

### Redis 키 구조

```
feed:main            → List (메인 피드 캐시, TTL 5분)
rank:articles        → Sorted Set (score=시간감쇠 점수, member=article_id)
                       -- ZINCRBY로 클릭/조회 시 점수 증가, 주기적 decay 적용
rank:tags            → Sorted Set (score=조회수, member=tag)
user:{id}:tags       → Set (유저 구독 태그)
```

> **원칙:** Redis는 캐시/랭킹 전용 휘발성 데이터만 저장.
> 아티클 원본 데이터의 소스 오브 트루스는 반드시 PostgreSQL + Elasticsearch.

### PostgreSQL — 관계형

```sql
users (id, email, fcm_token, created_at)
subscriptions (user_id, tag)
bookmarks (user_id, article_id, created_at)
push_logs (id, user_id, article_id, sent_at, type)
```

---

## API 설계

### Articles
```
GET  /articles?q=&tags=&page=     -- 전문 검색 (Elasticsearch, edge-ngram 자동완성)
GET  /articles/trending           -- 인기 아티클 (Redis ZSet 시간감쇠 점수)
GET  /articles/:id                -- 상세 조회 + 조회 이벤트 → Redis ZINCRBY
GET  /articles/:id/related        -- "More like this" (Elasticsearch MLT)
```

### User
```
POST /auth/signup
POST /auth/login
GET  /users/me
PUT  /users/me/tags               -- 구독 태그 설정
GET  /users/me/bookmarks
POST /users/me/bookmarks/:id
```

### Events
```
POST /events                      -- 유저 이벤트 수집 (배치)
```

### Push
```
POST /push/subscribe              -- FCM 토큰 등록
POST /push/test                   -- 테스트 발송
```

---

## 아티클 수집 → 푸시 흐름

```
node-cron (15분 간격)
  → RSS/API 크롤링 (dev.to, HN, Medium)
  → Elasticsearch 인덱싱
  → Redis Pub/Sub 이벤트 발행 ('new_article', tags)
  → Subscriber: 구독 태그 매칭 (user:{id}:tags)
  → FCM 푸시 발송 (firebase-admin)
  → PostgreSQL push_logs 기록
  → TimescaleDB 이벤트 기록
```

---

## 구현 단계

### Phase 1 — 백엔드 기반 + 검색 (난이도: M, ~3일)
- [ ] NestJS 프로젝트 셋업 + Docker Compose (ES + Redis + TimescaleDB + PG)
- [ ] Elasticsearch 아티클 인덱스 생성
- [ ] 아티클 CRUD API + 전문 검색 엔드포인트
- [ ] RSS 크롤러 기본 구현 (dev.to, Hacker News)

### Phase 2 — 캐시 & 랭킹 (난이도: S, ~1일)
- [ ] Redis 피드 캐시 (메인 피드 TTL)
- [ ] 인기 아티클 Sorted Set 업데이트 로직
- [ ] 자동완성 API (Redis Completion)

### Phase 3 — 유저 & 인증 (난이도: M, ~2일)
- [ ] 유저 회원가입/로그인 (JWT)
- [ ] 구독 태그 관리
- [ ] 북마크 기능

### Phase 4 — 푸시 알림 (난이도: M, ~2일)
- [ ] FCM 셋업 (firebase-admin)
- [ ] FCM 토큰 등록 API
- [ ] Redis Pub/Sub → 구독자 매칭 → FCM 발송 파이프라인
- [ ] 주간 트렌드 예약 푸시 (node-cron)

### Phase 5 — 모바일 앱 (난이도: L, ~5일)
- [ ] React Native (Expo) 프로젝트 셋업
- [ ] 피드 화면, 검색 화면, 북마크 화면
- [ ] 태그 구독 설정
- [ ] 푸시 알림 수신 처리
- [ ] 오프라인 북마크 (AsyncStorage)

### Phase 6 — 이벤트 분석 (난이도: M, ~2일)
- [ ] TimescaleDB 유저 이벤트 수집
- [ ] 앱/백엔드 이벤트 트래킹 추가
- [ ] 트렌드 분석 쿼리 작성
- [ ] (선택) devops-monitor Grafana에 분석 대시보드 연동

---

## 리스크 & 대응

| 리스크 | 대응 |
|--------|------|
| ES 메모리 사용량 높음 | JVM Heap 512m으로 제한 (`ES_JAVA_OPTS`) |
| 크롤링 차단 | User-Agent 설정, 요청 간격 조절, 공식 API 우선 사용 |
| FCM 토큰 만료 | 로그인 시마다 토큰 갱신, 만료 토큰 자동 정리 |
| 개인화 알림 과다 발송 | 태그별 알림 토글 + Digest 모드 (즉시/일간/주간 선택) |
| 크롤링 중복 아티클 | URL 해싱으로 인덱싱 전 중복 체크 |
| ES 인덱스 무한 증가 | ILM (Index Lifecycle Management) — 90일 이상 아티클 cold 티어 이동 |

---

## devops-monitor 연동 포인트

TechFeed 백엔드에 `/metrics` 엔드포인트 추가 → devops-monitor Prometheus가 수집.
유저 이벤트 → TimescaleDB → devops-monitor Grafana User Analytics 대시보드.
두 프로젝트가 실제로 연결되는 포트폴리오 구조.

---

## 참고 소스

- dev.to API: `https://dev.to/api/articles`
- Hacker News API: `https://hacker-news.firebaseio.com/v0/`
- Medium RSS: `https://medium.com/feed/tag/{tag}`
