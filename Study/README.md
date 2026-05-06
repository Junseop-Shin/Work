# Study Index

> 프로젝트에서 사용 중인 기술스택을 대화형으로 학습·정리하는 공간.
> `/study` 커맨드로 Q&A 후 카테고리에 맞춰 문서 추가.

---

## 학습 현황

### 1. 언어

| 기술 | 문서 | 상태 |
|------|------|------|
| TypeScript | [종합정리](언어/TypeScript-종합정리.md) | ✅ |
| Python | — | ⬜ |

<details>
<summary>TypeScript 버전별 follow-up</summary>

| 버전 | 문서 | 상태 |
|------|------|------|
| 5.x 신규 기능 | — | ⬜ |
| 6.0 | — | ⬜ |
| 7.0 (Go Native) | — | ⬜ |

</details>

### 2. 프론트엔드 프레임워크

| 기술 | 문서 | 상태 |
|------|------|------|
| React | [18+19 주요 변경사항](프론트엔드/React-18-19-주요변경사항.md) | 🟨 |
| Next.js | — | ⬜ |

<details>
<summary>React 버전별 follow-up</summary>

| 버전 | 주요 기능 | 문서 | 상태 |
|------|----------|------|------|
| 18 + 19 | Concurrent Rendering, Server Components, Compiler, Actions | [문서](프론트엔드/React-18-19-주요변경사항.md) | ✅ |
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

| 기술 | 문서 | 상태 |
|------|------|------|
| React Native / Expo | [RN vs Flutter](모바일/RN-vs-Flutter-비교.md), [Expo WebView 래퍼](모바일/RN-Expo-WebView래퍼-아키텍처.md) | 🟨 |
| expo-router | — | ⬜ |

<details>
<summary>RN follow-up</summary>

| 주제 | 문서 | 상태 |
|------|------|------|
| New Architecture (Fabric / TurboModules) | — | ⬜ |
| Hermes 엔진 | — | ⬜ |

</details>

### 4. 상태관리

| 기술 | 문서 | 상태 |
|------|------|------|
| Zustand | — | ⬜ |
| TanStack Query (React Query) | — | ⬜ |

### 5. UI / 스타일링

| 기술 | 문서 | 상태 |
|------|------|------|
| Tailwind CSS | — | ⬜ |
| Radix UI | — | ⬜ |
| shadcn/ui | — | ⬜ |
| styled-components | — | ⬜ |
| Motion (Framer Motion) | — | ⬜ |
| class-variance-authority (CVA) | — | ⬜ |

### 6. 폼 / 유효성 검사

| 기술 | 문서 | 상태 |
|------|------|------|
| React Hook Form | — | ⬜ |
| Pydantic | — | ⬜ |
| class-validator | — | ⬜ |

### 7. 백엔드 프레임워크

| 기술 | 문서 | 상태 |
|------|------|------|
| NestJS (Node.js) | — | ⬜ |
| FastAPI (Python) | — | ⬜ |

### 8. ORM / ODM

| 기술 | 문서 | 상태 |
|------|------|------|
| TypeORM | — | ⬜ |
| Mongoose | — | ⬜ |
| SQLAlchemy | — | ⬜ |

### 9. 인증

| 기술 | 문서 | 상태 |
|------|------|------|
| Passport.js / JWT | — | ⬜ |

### 10. 큐 / 스케줄링

| 기술 | 문서 | 상태 |
|------|------|------|
| BullMQ | — | ⬜ |
| Celery | — | ⬜ |

### 11. DB

| 기술 | 문서 | 상태 |
|------|------|------|
| PostgreSQL / TimescaleDB | — | ⬜ |
| MongoDB | — | ⬜ |
| Redis | — | ⬜ |
| Elasticsearch | — | ⬜ |

### 12. 시각화 / 차트

| 기술 | 문서 | 상태 |
|------|------|------|
| Three.js | — | ⬜ |
| Recharts | — | ⬜ |
| lightweight-charts (TradingView) | — | ⬜ |

### 13. 테스트

| 기술 | 문서 | 상태 |
|------|------|------|
| Vitest | — | ⬜ |
| Testing Library | — | ⬜ |
| Playwright | — | ⬜ |
| Storybook | — | ⬜ |

### 14. 번들러 / 빌드

| 기술 | 문서 | 상태 |
|------|------|------|
| Vite | — | ⬜ |

### 15. 인프라 / CI / 배포

| 기술 | 문서 | 상태 |
|------|------|------|
| Docker / Docker Compose | — | ⬜ |
| GitHub Actions | — | ⬜ |
| Cloudflare Tunnel | — | ⬜ |

### 16. 모니터링

| 기술 | 문서 | 상태 |
|------|------|------|
| Prometheus / Grafana | — | ⬜ |
| Loki | — | ⬜ |

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

- 폴더 구조: `Study/<카테고리>/<주제>.md`
- 카테고리는 위 학습 현황의 16개 분류를 따름 (예: `언어/`, `프론트엔드/`, `모바일/`)
- 파일명: `<주제>.md` (날짜 미포함, 변경 이력은 git이 관리)
- 학습 완료 시 위 표의 `문서` 칸에 링크, `상태` 칸을 ✅로 업데이트
- 버전별 follow-up은 해당 카테고리의 `<details>` 블록에 추가
