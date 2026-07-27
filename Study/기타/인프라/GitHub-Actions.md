# GitHub Actions — CI/CD 워크플로우

> 2026-06-08 | GitHub Actions, CI/CD, OIDC, 재사용워크플로우, Build Once

## 한 줄 요약

GitHub Actions는 **workflow→job(병렬)→step(순차)→runner** 4계층으로 자동화를 구성하며, "**Build Once, Deploy Many**(이미지 1회 빌드 후 환경 사이로 승격)"와 "**재사용 워크플로우**(로직은 한 곳, repo는 호출만)"가 잘 짜인 파이프라인의 두 축이다.

## 핵심 개념

### 구조 — 4계층

```
Workflow (.github/workflows/*.yml)   ← 트리거로 실행
 └─ Job (기본 병렬, 각자 새 runner)
     └─ Step (순차, run 또는 uses)
         └─ Runner (실행 머신)
```

```yaml
on:
  push: { branches: [main] }
  pull_request:
  workflow_dispatch:                 # 수동 실행
  schedule: [{ cron: '0 0 * * *' }]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm test
  deploy:
    needs: test                      # test 끝나야 시작(의존성)
    runs-on: ubuntu-latest
    steps: [...]
```

- **job은 기본 병렬**, 각자 **깨끗한 새 VM**(job 간 파일 공유 X → cache/artifact 필요).
- **step은 순차**, 같은 runner 작업공간 공유. 순서 강제는 `needs:`.

### 캐싱 vs 아티팩트 (용도가 다름)

| | `actions/cache` | `upload/download-artifact` |
|---|---|---|
| 목적 | 의존성·빌드 캐시 **재사용(속도)** | job 간/실행 후 **산출물 전달** |
| 예 | `~/.npm`, `node_modules` | 빌드된 `dist/`, 리포트 |

`setup-node`의 `cache: npm`처럼 setup 액션이 캐시를 자동 처리해주는 경우가 많다.

### 매트릭스 — 조합 병렬

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20, 22]
runs-on: ${{ matrix.os }}    # 2×3 = 6 job 병렬
```

### 시크릿 & 권한 — OIDC가 핵심 진화

```yaml
permissions:
  contents: read              # 최소 권한으로 좁히고
  packages: write             # 필요한 것만 열기
steps:
  - run: deploy.sh
    env: { API_KEY: ${{ secrets.API_KEY }} }   # 로그 자동 마스킹
```

**OIDC — 클라우드 비밀키를 저장하지 않는 방식(요즘 표준):**
장수명 액세스키를 시크릿에 박으면 유출 시 치명적. OIDC는 단기 토큰을 그때그때 발급.

```yaml
permissions: { id-token: write }
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/gha-deploy
      aws-region: ap-northeast-2
      # 액세스키 없음! GitHub OIDC 토큰을 AWS가 신뢰관계로 검증 → 임시 자격증명
```

GitHub이 "이 워크플로는 진짜 너의 repo"임을 OIDC로 증명 → 클라우드가 트러스트 정책으로 검증 → 임시 크레덴셜. **저장된 비밀키 0개.**

### Build Once, Deploy Many (핵심 원칙)

**빌드는 한 번만, 산출물(이미지)을 환경 사이로 승격**한다.

```
build(1회) → [같은 이미지] → deploy:staging → smoke → deploy:prod
                ↑ 환경마다 재빌드 금지
```

이유: ① 환경별 재빌드는 "테스트한 것 ≠ 배포한 것" 위험 ② 시크릿을 이미지에 구우면 유출(레이어 까보면 보임) ③ N환경 = N빌드 = 드리프트.

구현: build job이 이미지를 **불변 태그(커밋SHA)** 로 레지스트리에 push → deploy job들은 그 태그를 pull만.

#### 빌드 타임 vs 런타임 설정 (자주 부딪히는 지점)

"환경변수가 다르니 빌드도 따로?" → **대부분 런타임 주입이 정답.** DB 접속/시크릿/플래그/URL은 컨테이너 시작 시 env로 주입 → 이미지는 하나.

⚠️ **예외: 프론트엔드 클라이언트 변수**(`NEXT_PUBLIC_*`, `VITE_*`)는 브라우저에서 도므로 **빌드 시 번들에 인라인**된다. 그래도 Build-Once를 지키려면:
- **런타임 config 주입**(`/config.js`로 `window.__ENV__` 생성), 엔트리포인트 `envsubst` 치환, config 엔드포인트 fetch.
- Next.js는 서버 컴포넌트/SSR이 런타임 env를 그대로 읽음(빌드 박제는 `NEXT_PUBLIC_*`만).

### 최적화된 파이프라인 예시

```yaml
concurrency:                          # 같은 브랜치 연타 시 이전 실행 취소
  group: deploy-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build:
    outputs: { image: ${{ steps.meta.outputs.image }} }
    steps:
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha          # 레이어 캐시
          cache-to: type=gha,mode=max
      - id: meta
        run: echo "image=ghcr.io/${{ github.repository }}:${{ github.sha }}" >> $GITHUB_OUTPUT
  deploy-staging:
    needs: [build, test]
    environment: staging
    steps:
      - run: ./deploy.sh staging ${{ needs.build.outputs.image }}
      - run: ./smoke-test.sh https://staging.example.com
  deploy-prod:
    needs: deploy-staging
    environment: production             # 수동 승인 게이트
    steps:
      - run: echo "PREV=$(./current-image.sh prod)" >> $GITHUB_ENV   # 이전 버전 기록
      - run: ./deploy.sh prod ${{ needs.build.outputs.image }}        # 동일 이미지 승격
      - run: ./healthcheck.sh https://example.com --retries 10
      - if: failure()                    # 헬스체크 실패 시 롤백
        run: ./deploy.sh prod $PREV
```

### 재실행 단위 = job (중요)

GitHub "Re-run failed jobs"는 **실패 job + 의존 job만** 재실행, 성공 job 결과는 재사용. ⚠️ **최소 단위는 step이 아니라 job** → "재시도 경계를 job으로 쪼개라"(build/test/deploy를 별도 job으로). step 재시도는 `nick-fields/retry` 액션.

### 롤백 전략

| 전략 | 방식 |
|------|------|
| 이전 이미지 재배포 | 실패 시 직전 SHA로 재배포(위 예시) |
| 플랫폼 네이티브 | `kubectl rollout undo` / Cloud Run 이전 revision 트래픽 전환 |
| Blue-Green | 새 버전 띄워두고 LB 전환, 실패 시 즉시 복귀 |
| Canary | 트래픽 점진 전환, 지표 나쁘면 중단 |

핵심: **헬스체크 게이트 + 이전 버전 보존**이 있어야 롤백이 자동화된다.

### 여러 repo 공용 워크플로우 — 재사용 워크플로우

"하나 만들어 여러 repo가 공유" = `workflow_call`로 정의 + `uses`로 호출.

```yaml
# 공용 repo: .github/workflows/deploy.yml
on:
  workflow_call:
    inputs:
      service: { required: true, type: string }
      environment: { required: true, type: string }
    secrets:
      DEPLOY_TOKEN: { required: true }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps: [...]
```

```yaml
# 각 repo: 얇은 caller (job-level uses)
jobs:
  call:
    uses: my-org/shared-workflows/.github/workflows/deploy.yml@v1
    with: { service: techfeed-api, environment: production }
    secrets: inherit            # org/repo 시크릿 통째 전달
```

- 참조: `owner/repo/.github/workflows/파일.yml@ref`. **`@ref`는 태그(@v1)로 버저닝**(@main은 변경 즉시 전파라 위험).
- ⚠️ 공용 repo가 private이면 **Settings → Actions → Access**에서 조직 접근 허용해야 호출됨(자주 막힘).
- 차이는 **inputs**, 시크릿은 **org-level + `secrets: inherit`**.

#### 공유 메커니즘 4종

| 방법 | 공유 단위 | 위치 | 용도 |
|------|----------|------|------|
| Reusable workflow | job/파이프라인 | `jobs.x.uses` | 배포 흐름 전체 공유 |
| Composite action | step 묶음 | `steps.uses` | setup 등 step 조각 공유 |
| Required workflow(ruleset) | 강제 적용 | org 설정 | repo opt-in 없이 조직 강제 |
| Starter workflow | 템플릿 복사 | repo 생성 시 | 시작 템플릿 |

→ 프론트/백은 빌드·배포 메커니즘이 달라 **템플릿 자체를 2종**(frontend/backend)으로 두는 게 보통. 모노레포면 `paths:` 필터로 바뀐 쪽만 빌드.

### 함정 (보안 사고 단골)

- ⚠️ **`pull_request_target`** — 포크 PR이 시크릿에 접근 가능 + base 워크플로로 실행 → PR 코드를 checkout해 실행하면 시크릿 탈취.
- ⚠️ **시크릿을 로그로 가공**(`echo`) → 마스킹 우회.
- ⚠️ **신뢰 못 할 입력 보간** — PR 제목/브랜치명을 `run`에 직접 넣으면 스크립트 인젝션. `env:`로 받아라.
- ⚠️ **서드파티 액션** — 민감하면 태그 대신 **커밋 SHA로 고정**.

## 핵심 질의응답

**Q. 환경변수가 다르면 빌드도 따로 해야 하나?**
A. 대부분 아니다. 설정은 런타임 env 주입이 원칙이고 이미지는 하나(Build Once). 예외는 프론트 클라이언트 변수(빌드 인라인)인데, 그것도 런타임 config 주입으로 Build-Once를 지키는 게 모범.

**Q. 실패한 단계만 재실행 가능한가?**
A. 재실행 최소 단위는 step이 아니라 job이다. 그래서 재시도하고 싶은 경계를 별도 job으로 쪼개야 한다.

**Q. 여러 repo가 워크플로우 하나를 공유하려면?**
A. `on: workflow_call`로 재사용 워크플로우를 정의하고 각 repo가 `uses: org/repo/.github/workflows/x.yml@v1`로 호출. 차이는 inputs, 시크릿은 org-level + `secrets: inherit`. private이면 Access 허용 필수.

## 주의사항 / 자주 하는 실수

- **환경별 재빌드** — Build Once 위반. 동일 이미지를 승격하라.
- **장수명 클라우드 키 저장** — OIDC로 대체.
- **재사용 워크플로우 `@main` 참조** — 변경이 전 repo에 즉시 전파. 태그로 고정.
- **`pull_request_target`에서 PR 코드 실행** — 시크릿 탈취 경로.

## 참고

- [Docker · 컨테이너](Docker-컨테이너.md) — 빌드 대상 이미지
- [12/15-Factor App](../아키텍처/12-15-Factor-App.md) — Build/Release/Run 분리, config 원칙
- [GitHub Actions 공식 문서](https://docs.github.com/actions)
