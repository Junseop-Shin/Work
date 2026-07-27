# 12-Factor & 15-Factor App

> 2026-06-08 | 클라우드네이티브, 방법론, 12factor, stateless, 관측가능성

## 한 줄 요약

12-Factor App은 "클라우드/SaaS 앱이 이식성·확장성·자동화에 강하려면 지켜야 할 12개 원칙"이며, 핵심은 **설정 분리(③)·빌드와 실행 분리(⑤)·stateless 프로세스(⑥)·로그 스트림(⑪)** — 이후 15-Factor가 **API 우선·관측가능성·보안**을 보강했다.

## 핵심 개념

### 배경

Heroku가 2011년 정리한 방법론. 목표는 **이식성 있고(어디든 배포) · 확장 가능하고(scale-out) · 일회성/자동화에 강한** 앱. 오늘날 "클라우드 네이티브의 기본기"로 통한다.

### 12-Factor (주제별)

#### 코드 & 의존성
1. **Codebase** — 앱 하나 = 버전관리되는 코드베이스 하나, 여러 환경으로 배포.
2. **Dependencies** — 의존성을 명시적으로 선언·격리(시스템 암묵 의존 금지). → `package.json` + Docker.
3. **Config** — 환경마다 다른 설정은 **코드가 아니라 환경변수**에. (시크릿을 이미지에 박지 마라)
4. **Backing Services** — DB·Redis·큐·외부 API를 **교체 가능한 리소스**로(URL로 연결). 로컬↔클라우드 교체가 코드 변경 없이.

#### 빌드 & 실행
5. **Build, Release, Run** — 빌드(코드→산출물)/릴리스(산출물+설정)/실행을 엄격히 분리. → **Build Once, Deploy Many**, 릴리스는 불변(롤백=이전 릴리스).
6. **Processes** — 앱은 **stateless 프로세스**, 상태는 외부 저장소로. (메모리/로컬디스크 세션 금지 → scale-out 시 깨짐)
7. **Port Binding** — 앱이 스스로 포트를 열어 노출(외부 웹서버 의존 X).
8. **Concurrency** — 프로세스를 여러 개 띄워(**scale-out**) 확장. scale-up이 아니라.
9. **Disposability** — 빠르게 켜지고 **우아하게 종료**(graceful shutdown, SIGTERM 처리). 오토스케일·롤링의 전제.

#### 운영 & 환경
10. **Dev/Prod Parity** — dev/staging/prod를 최대한 비슷하게. → Docker가 푸는 문제.
11. **Logs** — 로그를 파일로 쓰지 말고 **stdout 이벤트 스트림**으로. 수집·저장은 인프라가(→ Loki).
12. **Admin Processes** — 마이그레이션 등 일회성 작업은 앱과 같은 환경의 일회성 프로세스로.

### 15-Factor — "Beyond the Twelve-Factor App" (Kevin Hoffman, 2016)

12개를 재정리하며 **3개 추가**. ※ 공식 후속이 아니라 가장 유명한 확장판. 기준선은 여전히 12-Factor.

**추가된 3개:**

13. **API First** — 코드보다 **API 계약(스펙)을 먼저** 설계하고 구현. OpenAPI로 계약을 못 박아 프론트·백 병렬 개발 + 계약 테스트. → [프론트-백엔드 계약 동기화](프론트-백엔드-계약-동기화.md)와 직결.

14. **Telemetry (관측가능성)** — 앱이 자기 상태를 밖으로 내보내야("볼 수 없으면 관리할 수 없다"). 3종: ① APM(성능지표→Prometheus) ② 도메인 로그/이벤트 ③ 헬스/시스템. → metrics/logs/traces 3기둥, 모니터링과 직결.

15. **Authentication & Authorization (보안)** — 인증(누구냐)·인가(뭘 할 수 있냐)를 처음부터 내장. AuthN + AuthZ(RBAC) + OAuth/OIDC. → [인증 종합](../인증/인증-JWT-OAuth-OIDC.md), Zero Trust와 직결.

**조정된 기존 factor:**
- "Build, Release, Run" → **"Design, Build, Release, Run"** (Design 추가).
- "Config" → **"Configuration, Credentials, and Code"** (시크릿을 코드와 섞지 마라 명시).

### 전체 표 (15-Factor)

| # | Factor | 비고 |
|---|--------|------|
| 1 | One Codebase, One App | |
| 2 | **API First** | 🆕 |
| 3 | Dependency Management | |
| 4 | **Design**, Build, Release, Run | Design 추가 |
| 5 | Config, **Credentials**, Code | 시크릿 강조 |
| 6 | Logs | |
| 7 | Disposability | |
| 8 | Backing Services | |
| 9 | Environment Parity | |
| 10 | Administrative Processes | |
| 11 | Port Binding | |
| 12 | Stateless Processes | |
| 13 | Concurrency | |
| 14 | **Telemetry** | 🆕 |
| 15 | **Authentication & Authorization** | 🆕 |

## 핵심 질의응답

**Q. 12-Factor의 핵심을 더 줄이면?**
A. ③설정 분리 + ⑤빌드/실행 분리 + ⑥stateless + ⑪로그 스트림. 인프라(Docker/CI/CD) 설계를 관통하는 4개.

**Q. 15-Factor가 12-Factor의 공식 후속인가?**
A. 아니다. Kevin Hoffman의 유명한 확장판이고, 업계 기준선은 여전히 원조 12-Factor다. 다만 추가된 API First/Telemetry/보안은 현대 클라우드 앱에 중요.

**Q. "환경변수가 다르니 빌드도 따로"가 12-Factor 위반인 이유?**
A. ③Config(설정은 환경변수) + ⑤Build/Release/Run 분리 위반. 설정은 런타임 주입하고 빌드 산출물은 하나여야 한다(Build Once).

## 주의사항 / 자주 하는 실수

- **설정/시크릿을 코드·이미지에 하드코딩** — ③⑤ 위반, 유출 위험.
- **프로세스에 상태 보관**(로컬 세션/파일) — ⑥ 위반, scale-out 시 깨짐.
- **로그를 파일로 직접 관리** — ⑪ 위반. stdout으로 흘려보내고 수집은 인프라가.
- **15-Factor를 표준으로 단정** — 확장판임을 인지.

## 참고

- [프론트-백엔드 계약 동기화](프론트-백엔드-계약-동기화.md) — API First
- [인증 종합 (JWT/OAuth/OIDC)](../인증/인증-JWT-OAuth-OIDC.md) — AuthN/AuthZ
- [Docker · 컨테이너](../인프라/Docker-컨테이너.md) / [GitHub Actions](../인프라/GitHub-Actions.md) / [Cloudflare](../인프라/Cloudflare.md)
- [The Twelve-Factor App](https://12factor.net/)
