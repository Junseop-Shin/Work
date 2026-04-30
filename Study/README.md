# Study Index

> 프로젝트에서 사용 중인 기술스택을 하나씩 정리하는 공간.
> `/study` 커맨드로 대화하며 학습 후 각 문서 추가.

---

## 학습 현황

| 기술 | 카테고리 | 사용 프로젝트 | 문서 | 상태 |
|------|----------|--------------|------|------|
| TypeScript | 언어 | 전체 | — | ⬜ |
| React 19 | 프론트엔드 | techfeed, kis-trader, boldgobynd, profile | — | ⬜ |
| Next.js 15 | 프론트엔드 | techfeed, kis-trader, boldgobynd, profile | — | ⬜ |
| React Native / Expo | 모바일 | techfeed-mobile | [비교](2026-04-29-RN-vs-Flutter-비교.md), [아키텍처](2026-04-29-RN-Expo-WebView래퍼-아키텍처.md) | 🟨 |
| Zustand | 상태관리 | techfeed-mobile, kis-trader | — | ⬜ |
| TanStack Query (React Query) | 상태관리/서버상태 | techfeed-mobile, kis-trader, my-ui-lib | — | ⬜ |
| Tailwind CSS | 스타일링 | kis-trader, boldgobynd, my-ui-lib | — | ⬜ |
| Radix UI | UI 컴포넌트 | kis-trader, my-ui-lib | — | ⬜ |
| shadcn/ui | UI 컴포넌트 | kis-trader | — | ⬜ |
| NestJS | 백엔드 | techfeed-api | — | ⬜ |
| TypeORM | ORM | techfeed-api | — | ⬜ |
| Mongoose | ODM | techfeed-api | — | ⬜ |
| BullMQ | 큐/작업 스케줄링 | techfeed-api, techfeed-crawler | — | ⬜ |
| Passport.js / JWT | 인증 | techfeed-api | — | ⬜ |
| FastAPI | 백엔드 (Python) | kis-trader, lotto-oracle | — | ⬜ |
| SQLAlchemy | ORM (Python) | kis-trader | — | ⬜ |
| Celery | 태스크 큐 (Python) | kis-trader | — | ⬜ |
| Pydantic | 데이터 검증 (Python) | kis-trader, lotto-oracle | — | ⬜ |
| Three.js | 3D 렌더링 | lotto-oracle | — | ⬜ |
| Recharts | 차트 | my-ui-lib, kis-trader | — | ⬜ |
| React Hook Form | 폼 | my-ui-lib | — | ⬜ |
| Motion (Framer Motion) | 애니메이션 | boldgobynd, profile, my-ui-lib | — | ⬜ |
| Storybook | UI 개발/문서화 | my-ui-lib | — | ⬜ |
| Vitest | 테스트 | my-ui-lib, lotto-oracle | — | ⬜ |
| Testing Library | 테스트 | my-ui-lib | — | ⬜ |
| Playwright | E2E 테스트 | my-ui-lib, lotto-oracle | — | ⬜ |
| PostgreSQL / TimescaleDB | DB | techfeed, kis-trader, devops-monitor | — | ⬜ |
| MongoDB | DB | techfeed | — | ⬜ |
| Redis | 캐시/큐 | techfeed, kis-trader | — | ⬜ |
| Elasticsearch | 검색 | techfeed | — | ⬜ |
| Docker / Docker Compose | 인프라 | 전체 | — | ⬜ |
| GitHub Actions | CI/CD | 전체 | — | ⬜ |
| Cloudflare Tunnel | 인프라/네트워킹 | 전체 | — | ⬜ |
| Prometheus / Grafana | 모니터링 | devops-monitor | — | ⬜ |
| Loki | 로그 수집 | devops-monitor | — | ⬜ |
| expo-router | 라우팅 (RN) | techfeed-mobile | — | ⬜ |
| class-variance-authority (CVA) | 스타일 유틸 | my-ui-lib | — | ⬜ |
| Vite | 번들러 | my-ui-lib, profile | — | ⬜ |
| Styled Components | 스타일링 | boldgobynd | — | ⬜ |

---

## 카테고리별 분류

### 언어
- TypeScript
- Python (FastAPI 생태계)

### 프론트엔드 프레임워크
- React 19
- Next.js 15/16
- React Native / Expo

### 상태관리
- Zustand — 클라이언트 전역 상태
- TanStack Query (React Query) — 서버 상태, 캐싱

### UI / 스타일링
- Tailwind CSS
- Radix UI — headless 컴포넌트 프리미티브
- shadcn/ui — Radix + Tailwind 래퍼
- styled-components
- Motion (Framer Motion)
- class-variance-authority (CVA)

### 백엔드
- NestJS (Node.js)
- FastAPI (Python)

### ORM / ODM
- TypeORM (NestJS + PostgreSQL)
- Mongoose (NestJS + MongoDB)
- SQLAlchemy (FastAPI + PostgreSQL)

### 인증
- Passport.js / JWT

### 큐 / 스케줄링
- BullMQ (Node.js)
- Celery (Python)

### DB
- PostgreSQL / TimescaleDB
- MongoDB
- Redis
- Elasticsearch

### 3D / 시각화
- Three.js
- Recharts
- lightweight-charts (TradingView)

### 폼 / 유효성 검사
- React Hook Form
- Pydantic
- class-validator (NestJS)

### 테스트
- Vitest
- Testing Library
- Playwright
- Storybook

### 번들러 / 빌드
- Vite
- Next.js (내장)

### 인프라
- Docker / Docker Compose
- GitHub Actions
- Cloudflare Tunnel
- Prometheus / Grafana / Loki

---

## 학습 우선순위 제안

면접 빈출 + 현재 프로젝트 핵심 기술 기준:

1. **TypeScript** — 전체 기반
2. **React 19** — 새 기능 (Server Components, Actions 등)
3. **Next.js 15/16** — App Router, RSC, 캐싱 전략
4. **TanStack Query** — 서버 상태 관리의 핵심
5. **Zustand** — 클라이언트 상태관리
6. **NestJS** — 아키텍처, DI, 데코레이터
7. **Radix UI / shadcn** — headless 컴포넌트 패턴
8. **Tailwind CSS** — 유틸리티 클래스 시스템
9. **React Hook Form** — 폼 상태 관리
10. **Three.js** — 3D 렌더링 기초

---

> 학습 완료 시 위 표의 `문서` 칸에 링크, `상태` 칸을 ✅로 업데이트
