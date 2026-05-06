# Study Index

> 프로젝트에서 사용 중인 기술스택을 대화형으로 학습·정리하는 공간.
> `/study` 커맨드로 Q&A 후 카테고리에 맞춰 문서 추가.

---

## 학습 현황

### 1. 언어

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| TypeScript | 전체 | [종합정리](2026-05-04-TypeScript-종합정리.md) | ✅ |
| Python | FastAPI 생태계 | — | ⬜ |

<details>
<summary>TypeScript 버전별 follow-up</summary>

| 버전 | 문서 | 상태 |
|------|------|------|
| 5.x 신규 기능 | — | ⬜ |
| 6.0 | — | ⬜ |
| 7.0 (Go Native) | — | ⬜ |

</details>

### 2. 프론트엔드 프레임워크

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| React | techfeed, kis-trader, boldgobynd, profile | — | ⬜ |
| Next.js | techfeed, kis-trader, boldgobynd, profile | — | ⬜ |

<details>
<summary>React 버전별 follow-up</summary>

| 버전 | 주요 기능 | 문서 | 상태 |
|------|----------|------|------|
| 19 | Server Components 안정화, Actions, `use` Hook | — | ⬜ |
| 20 | — | — | ⬜ |

</details>

<details>
<summary>Next.js 버전별 follow-up</summary>

| 버전 | 주요 기능 | 문서 | 상태 |
|------|----------|------|------|
| 15 | App Router, RSC, 캐싱 변경 | — | ⬜ |
| 16 | — | — | ⬜ |
| 17 | — | — | ⬜ |

</details>

### 3. 모바일

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| React Native / Expo | techfeed-mobile | [RN vs Flutter](2026-04-29-RN-vs-Flutter-비교.md), [Expo WebView 래퍼](2026-04-29-RN-Expo-WebView래퍼-아키텍처.md) | 🟨 |
| expo-router | techfeed-mobile | — | ⬜ |

<details>
<summary>RN follow-up</summary>

| 주제 | 문서 | 상태 |
|------|------|------|
| New Architecture (Fabric / TurboModules) | — | ⬜ |
| Hermes 엔진 | — | ⬜ |

</details>

### 4. 상태관리

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Zustand | techfeed-mobile, kis-trader | — | ⬜ |
| TanStack Query (React Query) | techfeed-mobile, kis-trader, my-ui-lib | — | ⬜ |

### 5. UI / 스타일링

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Tailwind CSS | kis-trader, boldgobynd, my-ui-lib | — | ⬜ |
| Radix UI | kis-trader, my-ui-lib | — | ⬜ |
| shadcn/ui | kis-trader | — | ⬜ |
| styled-components | boldgobynd | — | ⬜ |
| Motion (Framer Motion) | boldgobynd, profile, my-ui-lib | — | ⬜ |
| class-variance-authority (CVA) | my-ui-lib | — | ⬜ |

### 6. 폼 / 유효성 검사

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| React Hook Form | my-ui-lib | — | ⬜ |
| Pydantic | kis-trader, lotto-oracle | — | ⬜ |
| class-validator | techfeed-api | — | ⬜ |

### 7. 백엔드 프레임워크

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| NestJS (Node.js) | techfeed-api | — | ⬜ |
| FastAPI (Python) | kis-trader, lotto-oracle | — | ⬜ |

### 8. ORM / ODM

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| TypeORM | techfeed-api | — | ⬜ |
| Mongoose | techfeed-api | — | ⬜ |
| SQLAlchemy | kis-trader | — | ⬜ |

### 9. 인증

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Passport.js / JWT | techfeed-api | — | ⬜ |

### 10. 큐 / 스케줄링

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| BullMQ | techfeed-api, techfeed-crawler | — | ⬜ |
| Celery | kis-trader | — | ⬜ |

### 11. DB

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| PostgreSQL / TimescaleDB | techfeed, kis-trader, devops-monitor | — | ⬜ |
| MongoDB | techfeed | — | ⬜ |
| Redis | techfeed, kis-trader | — | ⬜ |
| Elasticsearch | techfeed | — | ⬜ |

### 12. 시각화 / 차트

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Three.js | lotto-oracle | — | ⬜ |
| Recharts | my-ui-lib, kis-trader | — | ⬜ |
| lightweight-charts (TradingView) | kis-trader | — | ⬜ |

### 13. 테스트

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Vitest | my-ui-lib, lotto-oracle | — | ⬜ |
| Testing Library | my-ui-lib | — | ⬜ |
| Playwright | my-ui-lib, lotto-oracle | — | ⬜ |
| Storybook | my-ui-lib | — | ⬜ |

### 14. 번들러 / 빌드

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Vite | my-ui-lib, profile | — | ⬜ |

### 15. 인프라 / CI / 배포

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Docker / Docker Compose | 전체 | — | ⬜ |
| GitHub Actions | 전체 | — | ⬜ |
| Cloudflare Tunnel | 전체 | — | ⬜ |

### 16. 모니터링

| 기술 | 사용 프로젝트 | 문서 | 상태 |
|------|--------------|------|------|
| Prometheus / Grafana | devops-monitor | — | ⬜ |
| Loki | devops-monitor | — | ⬜ |

---

## 학습 우선순위

면접 빈출 + 현재 프로젝트 핵심 기술 기준:

1. **TypeScript** — 전체 기반 (✅ 종합정리 완료, 버전별 follow-up 진행)
2. **React 19** — Server Components / Actions / `use` Hook
3. **Next.js 15/16** — App Router, RSC, 캐싱 전략
4. **TanStack Query** — 서버 상태 관리
5. **Zustand** — 클라이언트 상태관리
6. **NestJS** — 아키텍처, DI, 데코레이터
7. **Radix / shadcn** — headless 컴포넌트 패턴
8. **Tailwind CSS** — 유틸리티 클래스 시스템
9. **React Hook Form** — 폼 상태 관리
10. **Three.js** — 3D 렌더링 기초

---

## 문서 작성 규칙

- 파일명: `YYYY-MM-DD-<주제>.md`
- 학습 완료 시 위 표의 `문서` 칸에 링크, `상태` 칸을 ✅로 업데이트
- 버전별 follow-up은 해당 카테고리의 `<details>` 블록에 추가
