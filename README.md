# 🗂️ Work Space

> **개인 학습 · 프로젝트 · 경력 관리** 대시보드
> TIL(Today I Learned) 방식으로 기록하는 지식 저장소

---

## 📌 목차

| 폴더 | 설명 |
|------|------|
| [📚 Study](#-study) | 자격증 및 기술 학습 |
| [💻 Coding Tests](#-coding-tests) | 알고리즘 풀이 모음 |
| [🏢 Work History](#-work-history) | 경력 및 직무 기록 |
| [🚀 Projects](#-projects) | 개인 프로젝트 링크 허브 |
| [🏗️ Infrastructure](#-infrastructure) | 인프라 · 아키텍처 문서 |

---

## 📚 Study

> [Study 폴더 바로가기](./Study/) · 카테고리별 학습 현황은 [Study/README.md](./Study/README.md) 참조

`/study` 커맨드로 대화형 학습 후 카테고리별 문서로 정리합니다.

### 기술 학습

> 카테고리별 상세 현황은 [Study/README.md](./Study/README.md)

| 카테고리 | 주요 항목 |
|---------|----------|
| 언어 | TypeScript (✅ 종합정리), Python |
| 프론트엔드 프레임워크 | React (✅ 18+19 주요 변경사항), Next.js (✅ 기본, ✅ 15+16 주요 변경사항) |
| 모바일 | React Native / Expo (✅ RN vs Flutter, ✅ WebView 래퍼), expo-router |
| 상태관리 | (✅ 서버 상태 비교: TanStack/SWR/RTK Query), (✅ 클라이언트 상태 비교: Zustand/Redux/MobX/Valtio/Jotai) |
| UI / 스타일링 | Tailwind, Radix, shadcn, styled-components, Motion, CVA |
| 폼 / 유효성 | React Hook Form, Pydantic, class-validator |
| 백엔드 | NestJS, FastAPI |
| ORM / ODM | TypeORM, Mongoose, SQLAlchemy |
| 인증 / 큐 | Passport.js / JWT, BullMQ, Celery |
| DB | PostgreSQL/TimescaleDB, MongoDB, Redis, Elasticsearch |
| 시각화 / 차트 | Three.js, Recharts, lightweight-charts |
| 테스트 | Vitest, Testing Library, Playwright, Storybook |
| 인프라 / CI | Docker, GitHub Actions, Cloudflare Tunnel |
| 모니터링 | Prometheus, Grafana, Loki |

### 자격증

| 분야 | 항목 |
|------|------|
| DB | SQLP |
| Cloud | AZ-104 (Microsoft Azure Administrator) |

### 자격증 인증 사진 첨부 가이드

```
Study/
└── Certifications/
    ├── SQLP/
    │   ├── certificate.md        ← 합격 정보 + 인증 사진 임베드
    │   └── (이미지는 GitHub에 직접 업로드 후 링크)
    └── AZ-104/
        ├── certificate.md
        └── ...
```

**certificate.md 작성 예시:**

```markdown
## SQLP 합격 인증

- 취득일: YYYY-MM-DD
- 점수: OOO점

![SQLP 합격 인증서](https://github.com/YOUR_ID/Work/assets/ASSET_ID/image.png)
<!-- ※ 원본 파일은 ~/Documents/General/Certifications/ 에 보관 -->
<!-- ※ GitHub에 올리는 이미지는 점수·주민번호 등 민감 정보를 블러 처리할 것 -->
```

> **보안 주의사항**
> - 원본 인증서 파일은 `~/Documents/General/`에 보관 (AI 접근 금지 영역)
> - GitHub 업로드 시 개인정보(주민번호, 수험번호 등)는 반드시 모자이크/블러 처리
> - 이미지는 파일 직접 첨부 대신 GitHub Issue/PR에 드래그하여 URL만 사용 권장

---

## 💻 Coding Tests

> [Coding_Tests 폴더 바로가기](./Coding_Tests/)

알고리즘 풀이 및 코딩 테스트 대비 자료를 정리합니다.

---

## 🏢 Work History

> [Work_History 폴더 바로가기](./Work_History/)

경력 사항, 프로젝트 성과, 회고록을 기록합니다.

---

## 🚀 Projects

> [Projects 폴더 바로가기](./Projects/)

각 프로젝트는 **독립 Git 레포**로 관리됩니다. 링크 허브 역할만 합니다.

| 프로젝트 | 설명 | GitHub |
|---------|------|--------|
| profile | 개인 프로필 페이지 | [Junseop-Shin/profile](https://github.com/Junseop-Shin/profile) |
| boldgobynd | boldgobynd 프로젝트 | [Junseop-Shin/boldgobynd](https://github.com/Junseop-Shin/boldgobynd) |
| my-ui-lib | 개인 UI 컴포넌트 라이브러리 | [Junseop-Shin/my-ui-lib](https://github.com/Junseop-Shin/my-ui-lib) |
| Junseop-Shin | GitHub 프로필 README | [Junseop-Shin/Junseop-Shin](https://github.com/Junseop-Shin/Junseop-Shin) |
| kis-trader | KIS API 기반 자동 트레이딩 | [Junseop-Shin/kis-trader](https://github.com/Junseop-Shin/kis-trader) |
| EY-hy-2026 | EY 2026 관련 프로젝트 | [Junseop-Shin/EY-hy-2026](https://github.com/Junseop-Shin/EY-hy-2026) |

---

## 🏗️ Infrastructure

> [Infrastructure 폴더 바로가기](./Infrastructure/)

IaC, 서버 설정, 아키텍처 다이어그램 등을 관리합니다.

---

## 📋 Setup

셋팅 계획 및 진행 상황 → [setup_plan.md](./setup_plan.md)

---

<p align="center">
  <sub>🤖 AI-assisted workspace · Last updated: 2026-02-20</sub>
</p>
