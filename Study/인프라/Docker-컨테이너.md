# Docker · 컨테이너 — 이미지/Compose/클라우드 실행

> 2026-06-08 | Docker, 컨테이너, 멀티스테이지, Compose, Kubernetes

## 한 줄 요약

컨테이너는 VM처럼 OS를 통째로 가상화하는 게 아니라 **호스트 커널을 공유하면서 namespace(격리)·cgroup(자원제한)·union FS(레이어)로 가둔 프로세스**이며, 이 경량성 덕에 빠르게 띄우고 어디서나 동일하게 실행한다.

## 핵심 개념

### 왜 컨테이너인가 — VM과의 차이가 본질

```
┌─ VM ──────────────┐   ┌─ 컨테이너 ────────┐
│ 앱 / 라이브러리     │   │ 앱 / 라이브러리    │
│ ★ 게스트 OS 통째    │   │ (게스트 OS 없음)   │
│ ──하이퍼바이저──    │   │ ──도커 엔진──      │
│ 호스트 OS / HW     │   │ ★ 호스트 OS 커널 공유 │
└───────────────────┘   └───────────────────┘
```

| | VM | 컨테이너 |
|---|---|---|
| 격리 | 하드웨어 가상화 | **OS 커널 수준** |
| 게스트 OS | 각자 풀 OS(GB) | **없음, 커널 공유**(MB) |
| 부팅 | 수십 초 | **수백 ms** |
| 격리 강도 | 강함 | 상대적으로 약함(커널 공유) |

컨테이너는 마법이 아니라 **격리된 리눅스 프로세스**다. 3대 커널 기능:
1. **namespace (격리)** — PID/Network/Mount/User를 격리해 "너만의 세상"을 보여줌(컨테이너 안에선 자기 프로세스가 PID 1).
2. **cgroups (자원 제한)** — CPU/메모리/IO 상한 강제(`--memory 512m`).
3. **union filesystem (레이어)** — 이미지 레이어 공유.

> 커널을 공유하므로 리눅스 컨테이너는 리눅스 커널이 필요하다. 그래서 Mac/Windows의 Docker는 내부에 **리눅스 VM을 하나** 돌린다.

### 이미지 vs 컨테이너, 그리고 레이어

- **이미지 = 읽기전용 레이어들의 스택** / **컨테이너 = 그 위에 쓰기가능 레이어 1장을 얹은 실행 인스턴스**.
- **동일 이미지로 컨테이너 여러 개**를 띄울 수 있다(베이스 레이어는 디스크에서 공유).
- 컨테이너는 stateless — 쓰기 레이어는 컨테이너와 함께 소멸. 영속 데이터는 **볼륨**으로.
- **Dockerfile 한 줄 = 레이어 1장**, 그리고 레이어는 **빌드 시 캐시**된다.

⚠️ 멀티스테이지의 "스테이지"는 빌드 과정 개념이고, **최종 이미지는 마지막 스테이지의 레이어로만** 구성된다("스테이지로 구성된 이미지"는 부정확).

### 레이어 캐시 — 빌드 최적화의 핵심

규칙: 어떤 레이어가 바뀌면 **그 이후 레이어 캐시가 전부 무효화**된다. → 자주 안 바뀌는 걸 위로.

```dockerfile
# ✅ 의존성 안 바뀌면 install 레이어 캐시 재사용
COPY package*.json ./
RUN npm install        # 의존성 변화 없으면 캐시 hit
COPY . .               # 소스만 바뀜 → 여기부터만 재빌드
```

⚠️ **캐시는 `docker build`(빌드) 단계의 것.** `docker run`/`compose up`(실행)은 완성된 이미지로 컨테이너를 생성할 뿐 재빌드/캐싱이 아니다. `.dockerignore`로 `node_modules`·`.git`을 빌드 컨텍스트에서 제외.

### Dockerfile 최적화 — 멀티스테이지 & 경량화

```dockerfile
FROM node:20 AS builder        # 빌드용(무거움)
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-slim              # 런타임용(가벼움) — 결과물만 복사
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node                      # ⚠️ root 금지
CMD ["node", "dist/main.js"]
```

| 베이스 | 크기 | 특징 |
|--------|------|------|
| `node:20` | ~1GB | 빌드툴 다 있음. 빌드 단계용 |
| `node:20-slim` | ~200MB | 런타임 기본값 |
| `node:20-alpine` | ~50MB | musl libc → 네이티브 모듈 이슈 가능 |
| `distroless` | ~20MB | 셸 없음 → 공격면 최소, 디버깅 불편 |

작을수록 보안·전송·콜드스타트에 유리하나 호환성/디버깅을 희생.

### Docker Compose — 단일 호스트 멀티 컨테이너

```yaml
services:
  api:
    build: .
    depends_on:
      db: { condition: service_healthy }   # ⚠️ 핵심
    environment:
      DATABASE_URL: postgres://db:5432/app  # 'db'(서비스명)이 곧 호스트명
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
    volumes: [pgdata:/var/lib/postgresql/data]   # named volume = 영속
volumes:
  pgdata:
```

- **서비스명이 DNS 호스트명**이 된다(자동 생성 네트워크).
- ⚠️ **`depends_on`만으론 부족** — "컨테이너 떴다 ≠ DB가 쿼리 받을 준비됨". **healthcheck + `condition: service_healthy`** 로 진짜 순서 보장.
- named volume = 영속 / bind mount = 호스트 경로 직결(개발 핫리로드).
- **Compose는 단일 호스트·개발/소규모용** — 멀티노드·오토스케일·HA는 못 함.

### 클라우드 컨테이너 — "이미지를 어디서 돌리나"

```
손 많이 감 ←──────────────────────────────────→ 알아서 해줌
VM에 docker run → Compose → Managed(서버리스 컨테이너) → Kubernetes
```

- **0) 레지스트리**: 이미지를 push/pull. Docker Hub / GHCR / ECR / Artifact Registry / ACR.
- **1) Managed 서버리스 컨테이너** (요즘 기본 추천): 이미지만 주면 인프라를 클라우드가 관리, 오토스케일(0까지), 요청당 과금.
  - **Cloud Run**(GCP, 가장 간편) / **ECS+Fargate**(AWS) / **Container Apps**(Azure).
- **2) Kubernetes**: 멀티노드 클러스터 오케스트레이터. 셀프힐링·오토스케일·롤링·서비스디스커버리. 강력하지만 복잡. 대규모·다수 서비스용.
  - 매니지드 k8s: **EKS**(AWS) / **GKE**(GCP) / **AKS**(Azure).

| 상황 | 추천 |
|------|------|
| 로컬 개발 / 소규모 단일 서버 | Docker Compose |
| 트래픽 가변·운영 인력 적음 | Managed(Cloud Run/Fargate) |
| 서비스 다수·대규모·세밀한 제어 | Kubernetes |

> **K8s = Kubernetes**(K + 글자8개 + s)의 약어, 별도 도구가 아님. Compose와 k8s는 경쟁이 아니라 규모 단계가 다름(Docker Swarm은 사실상 사장).

### 헬스체크는 2종류 — liveness vs readiness

| | Liveness(살아있나) | Readiness(트래픽 받을 준비) |
|---|---|---|
| 질문 | 프로세스 죽었나? | 지금 요청 처리 가능? |
| 실패 시 | **재시작** | **LB에서 잠시 제외**(트래픽 차단) |
| 예 | 데드락 → 재시작 | DB연결/워밍업 중 → 준비 시까지 제외 |

Compose의 `service_healthy`, k8s 롤링 배포, 매니지드 LB가 다 **readiness** 개념. 무중단 배포의 토대.

## 핵심 질의응답

**Q. 컨테이너와 VM 차이는?**
A. VM은 하이퍼바이저로 하드웨어를 가상화해 게스트 OS를 통째로 올리고, 컨테이너는 호스트 커널을 공유하며 namespace/cgroup으로 격리한 프로세스다. 그래서 컨테이너가 훨씬 가볍지만 격리는 상대적으로 약하다.

**Q. 이미지는 스테이지로 구성되나?**
A. 아니다. 이미지는 레이어로 구성된다. 멀티스테이지는 빌드 기법이고 최종 이미지엔 마지막 스테이지 레이어만 남는다.

**Q. 캐시는 컨테이너 실행할 때 되나?**
A. 아니다. 레이어 캐시는 `docker build`(빌드) 단계의 것이다. `run`/`compose up`은 완성 이미지로 컨테이너를 생성하는 별도 단계다.

**Q. 매니지드 쓰면 로드밸런싱·커넥션스트링 자동인가?**
A. 앞단 트래픽은 고정 엔드포인트 + 자동 LB라 스케일해도 연결 설정을 바꿀 일이 없다. 단 DB 등 외부 의존성 접속 정보는 여전히 env/시크릿으로 한 번 주입한다.

## 주의사항 / 자주 하는 실수

- **root 유저로 실행** — `USER`로 비root 지정(보안).
- **`depends_on`만 믿기** — healthcheck 없이는 순서 보장 안 됨.
- **상태를 컨테이너에 저장** — 쓰기 레이어는 소멸. 볼륨/외부 저장소로.
- **Compose를 프로덕션 오케스트레이터로 착각** — 단일 호스트 한계. 규모 커지면 Managed/k8s.
- **빌드 컨텍스트 비대화** — `.dockerignore` 누락 시 느리고 캐시 깨짐.

## 참고

- [GitHub Actions](GitHub-Actions.md) — 이 이미지를 빌드/배포하는 CI/CD
- [Cloudflare](Cloudflare.md) — 엣지 컴퓨트/컨테이너 배포 대안
- [12/15-Factor App](../아키텍처/12-15-Factor-App.md) — stateless·dev/prod parity 원칙
- [Docker 공식 문서](https://docs.docker.com/)
