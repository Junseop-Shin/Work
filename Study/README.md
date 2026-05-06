# Study Index

> 프로젝트에서 사용 중인 기술스택을 대화형으로 학습·정리하는 공간.
> `/study` 커맨드로 Q&A 후 각 카테고리에 문서 추가.

---

## 학습 현황

### 1. TypeScript

| 주제 | 문서 | 상태 |
|------|------|------|
| 종합 정리 (기초 → 7.0 Native) | [2026-05-04](2026-05-04-TypeScript-종합정리.md) | ✅ |
| 5.x 신규 기능 follow-up | — | ⬜ |
| 6.0 follow-up | — | ⬜ |
| 7.0 (Go Native) follow-up | — | ⬜ |

### 2. React

| 주제 | 문서 | 상태 |
|------|------|------|
| 종합 정리 (기초 + Hooks) | — | ⬜ |
| 19 신규 기능 (Actions, use, RSC 안정화) | — | ⬜ |
| 20 follow-up | — | ⬜ |

### 3. Next.js

| 주제 | 문서 | 상태 |
|------|------|------|
| 종합 정리 (App Router, RSC, 캐싱) | — | ⬜ |
| 15 신규 기능 follow-up | — | ⬜ |
| 16 follow-up | — | ⬜ |
| 17 follow-up | — | ⬜ |

### 4. React Native

| 주제 | 문서 | 상태 |
|------|------|------|
| RN vs Flutter 비교 | [2026-04-29](2026-04-29-RN-vs-Flutter-비교.md) | ✅ |
| Expo WebView 래퍼 아키텍처 | [2026-04-29](2026-04-29-RN-Expo-WebView래퍼-아키텍처.md) | ✅ |
| New Architecture (Fabric/TurboModules) | — | ⬜ |
| expo-router 심화 | — | ⬜ |

### 5. 상태관리

| 주제 | 문서 | 상태 |
|------|------|------|
| Zustand — 클라이언트 전역 상태 | — | ⬜ |
| TanStack Query — 서버 상태 / 캐싱 | — | ⬜ |

### 6. 스타일링

| 주제 | 문서 | 상태 |
|------|------|------|
| Tailwind CSS — 유틸리티 클래스 시스템 | — | ⬜ |
| Radix UI / shadcn — headless 컴포넌트 패턴 | — | ⬜ |
| styled-components | — | ⬜ |
| Motion (Framer Motion) | — | ⬜ |
| class-variance-authority (CVA) | — | ⬜ |

---

## 학습 우선순위

면접 빈출 + 현재 프로젝트 핵심 기술 기준:

1. **TypeScript** — 전체 기반 (✅ 종합정리 완료, 버전별 follow-up 진행)
2. **React 19** — Server Components / Actions / `use` Hook
3. **Next.js 15/16** — App Router, RSC, 캐싱 전략
4. **TanStack Query** — 서버 상태 관리
5. **Zustand** — 클라이언트 상태관리
6. **Radix / shadcn** — headless 컴포넌트
7. **Tailwind CSS** — 유틸리티 클래스 시스템

---

## 문서 작성 규칙

- 파일명: `YYYY-MM-DD-<주제>.md`
- 학습 완료 시 위 표의 `문서` 칸에 링크, `상태` 칸을 ✅로 업데이트
- 새 카테고리 추가가 필요하면 이 README 상단 표 구조를 따를 것
