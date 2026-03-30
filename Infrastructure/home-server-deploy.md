# Home Server Deploy — 삽질 기록

> Windows 홈서버에 Next.js profile 앱을 GitHub Actions로 자동 배포하기까지의 전 과정.

---

## 최종 구성 요약

| 항목 | 내용 |
|------|------|
| 서버 | Windows 10 홈서버 (`DESKTOP-2AI7EKV`) |
| SSH 접근 | 외부 포트 2222 → 내부 22 (공유기 포트포워딩) |
| 도메인 | `windows.nuclearbomb6518.com` (Cloudflare DNS A레코드 + DDNS) |
| 파일 전송 | `tar \| ssh` 파이프 (rsync 대체) |
| 프로세스 관리 | pm2 + Windows Task Scheduler (`PM2-Resurrect`) |
| HTTP 서비스 | Cloudflare Tunnel (`profile.nuclearbomb6518.com → localhost:3000`) |

---

## 시도 1 — Cloudflare Tunnel SSH

### 계획
`cloudflared access ssh`로 GitHub Actions → Cloudflare → 홈서버 SSH 연결.

### 실패 원인
```
websocket: bad handshake
```
- `cloudflared access ssh`는 **Cloudflare Zero Trust Access Application** 설정이 필요
- 단순히 cloudflared tunnel만 설치한다고 SSH proxy가 되지 않음
- GitHub Actions 러너는 `~/.ssh/config`의 ProxyCommand를 그대로 쓸 수 없음 (로컬 환경 설정이므로)

### 시도했던 것들
- `cert.pem` base64 인코딩으로 secret 등록 후 복원
- CF-Access-Client-Id / CF-Access-Client-Secret 서비스 토큰 발급
- Cloudflare Zero Trust → Self-hosted Application 생성 시도 → SSH용이 아닌 Web 앱 타입
- Infrastructure Application 시도 → "at least one target is required" 오류
- `.ssh/config`에 ProxyCommand 추가 → Actions 러너에는 cloudflared 없음

### 결론
Zero Trust SSH는 VNET 구성 등 추가 셋업이 필요. 포기.

---

## 시도 2 — Tailscale

### 계획
Tailscale VPN으로 GitHub Actions 러너와 홈서버를 같은 네트워크에 연결.

### 결론
유료 플랜 또는 가입 필요. 즉시 포기.

---

## 시도 3 — 직접 SSH (DDNS)

### 계획
공유기 포트포워딩(2222 → 22)으로 직접 SSH. 동적 IP 문제는 Cloudflare DDNS로 해결.

### Cloudflare DDNS 설정 (Windows PowerShell)
`C:\ddns.ps1` — 5분마다 실행되는 Task Scheduler 작업:
1. `ipify.org`로 현재 공인 IP 조회
2. Cloudflare API로 `windows.nuclearbomb6518.com` A레코드 업데이트
3. DNS는 **Proxy OFF** (오렌지 클라우드 끔) — SSH 포트는 Cloudflare 프록시 불가

### 문제: Cloudflare 프록시가 포트 2222 차단
처음엔 CNAME으로 터널 연결 → Cloudflare 프록시 IP(172.67.x.x)로 해석 → 2222 포트 차단.
→ A레코드 + DNS-only로 변경.

### SSH 키 문제
```
Load key "id_deploy": error in libcrypto
```
- `echo "$SECRET"` 방식으로 저장 시 개행문자 깨짐
- 해결: 키를 **base64 인코딩**해서 secret 저장, Actions에서 `base64 -d`로 복원

### Windows OpenSSH 설정
- 공개키를 `~/.ssh/authorized_keys`가 아닌 `C:\ProgramData\ssh\administrators_authorized_keys`에 등록해야 함
- icacls 권한 설정 필요:
  ```
  icacls administrators_authorized_keys /inheritance:r
  icacls administrators_authorized_keys /grant SYSTEM:F
  icacls administrators_authorized_keys /grant Administrators:F
  ```

---

## 시도 4 — rsync

### 계획
SSH 연결 성공 후 rsync로 빌드 파일 전송.

### 실패 원인
```
rsync: unexplained error (code 255)
```
Windows에 rsync 없음.

### 해결
`tar | ssh` 파이프로 대체:
```bash
tar czf - -C next/.next/standalone . | ssh ... "tar xzf - -C /path/next"
```

---

## 시도 5 — Next.js 빌드 구조

### 문제
`node_modules`를 서버에 복사하면 용량이 너무 큼.

### 해결
`next.config.ts`에 `output: "standalone"` 추가.
- `.next/standalone/` 안에 최소한의 node_modules 포함
- `server.js` 하나로 실행 가능

---

## 시도 6 — tar 디렉토리 없음

### 오류
```
tar: could not chdir to .next/static
```

### 원인
서버에 디렉토리가 없는 상태에서 tar 추출 시도.

### 해결
tar 전에 명시적으로 디렉토리 생성:
```bash
$SSH "mkdir next && mkdir next\.next && mkdir next\.next\static && mkdir next\public"
```

---

## 시도 7 — pm2 cwd 문제

### 오류
```
Cannot find module 'C:\Users\ylswn\server.js'
```

### 원인
`ecosystem.config.js`에 `cwd: "./"` → pm2를 어디서 실행하느냐에 따라 경로가 달라짐.

### 해결
```js
cwd: __dirname  // ecosystem.config.js 위치 기준으로 절대경로 고정
```

---

## 시도 8 — pm2 데몬이 SSH 세션 종료 시 죽음

### 증상
SSH 세션 종료 후 새 세션에서 `pm2 list` → `Spawning PM2 daemon` (새로 시작됨).

### 원인
**Windows OpenSSH Job Object**: SSH 세션이 Job Object를 만들고, 세션 종료 시 Job Object 내 모든 프로세스 kill. pm2 daemon도 포함.

### 시도한 것들
- `pm2 startup` → Windows 미지원 (`Init system not found`)
- `pm2-windows-service` → deprecated, 대화형 입력 필요로 실패
- `pm2-windows-startup` → 레지스트리 등록만 함, 즉각적인 문제 미해결

### 해결: Windows Task Scheduler
```cmd
schtasks /create /tn "PM2-Resurrect" /tr "cmd /c cd /d C:\...\next && pm2 resurrect" /sc ONLOGON /ru ylswn /rl HIGHEST /f
```
- Task Scheduler는 **자체 프로세스 트리**에서 실행 → SSH Job Object 외부
- `schtasks /run /tn "PM2-Resurrect"` 트리거 → pm2 daemon이 Task Scheduler 트리에서 기동 → SSH 세션 꺼져도 생존

### 배포 workflow에 추가
```bash
# Reload pm2 step 마지막에 추가
schtasks /run /tn PM2-Resurrect
```

---

## 시도 9 — 파일 락 오류

### 오류
```
다른 프로세스가 파일을 사용 중이기 때문에 프로세스가 액세스 할 수 없습니다.
```
`rmdir` 실패 → `mkdir` 실패 (폴더 이미 존재).

### 원인
pm2가 실행 중일 때 Windows는 열린 파일 삭제/이동 불가 (Linux와 다름).

### 해결
파일 전송 전 pm2 먼저 stop:
```bash
$SSH "pm2 stop profile-next" || true
$SSH "rmdir /s /q ..."
```

---

## lotto-oracle (Docker) 배포 — 삽질 기록

### 배포 대상
- FastAPI + Playwright 기반 로또 번호 생성기
- Docker Compose로 실행, `lotto.nuclearbomb6518.com` → `localhost:3001`

---

### 문제 1 — Docker BuildKit 크리덴셜 오류

**오류**
```
error getting credentials - err: exit status 1, out: 'A specified logon session does not exist.'
```

**원인**
Docker Desktop이 `credsStore: "desktop"` 설정을 통해 Windows Credential Manager를 사용하는데,
SSH 세션에서 실행하면 로그인 세션이 없어 Credential Manager 접근 불가.

**해결**
`C:\Users\ylswn\.docker\config.json`에서 `credsStore` 제거, 빈 auth 명시:
```json
{
  "auths": {
    "https://index.docker.io/v1/": {}
  },
  "currentContext": "desktop-linux"
}
```
`C:\Users\ylswn\AppData\Roaming\Docker\daemon.json`에서 BuildKit 비활성화:
```json
{
  "features": { "buildkit": false }
}
```

---

### 문제 2 — apt-get 400 Bad Request

**오류**
```
E: Failed to fetch http://deb.debian.org/.../libpolkit-gobject... 400 Bad Request
```

**원인**
Docker Desktop 내부 허브 프록시(`http.docker.internal:3128`)가 컨테이너 내 모든 HTTP 트래픽을 가로채는데,
APT 패키지 요청까지 도커 허브 프록시로 통과시켜 400 오류 발생.

**해결**
`docker compose build` 시 프록시 build-arg를 빈 값으로 덮어씌움:
```bash
docker compose build \
  --build-arg HTTP_PROXY= \
  --build-arg HTTPS_PROXY= \
  --build-arg http_proxy= \
  --build-arg https_proxy=
```

---

### 문제 3 — playwright install --with-deps 실패

**오류**
```
process "/bin/sh -c playwright install chromium --with-deps" did not complete successfully: exit code 1
```

**원인**
`--with-deps` 옵션이 내부적으로 apt-get을 다시 실행 → 프록시 문제 재발.

**해결**
Dockerfile에서 chromium을 apt-get으로 먼저 설치하고, playwright는 `--with-deps` 없이 실행:
```dockerfile
RUN apt-get install -y chromium chromium-driver
RUN playwright install chromium   # --with-deps 제거
```

---

### 문제 4 — ModuleNotFoundError: No module named 'src'

**오류**
```
ModuleNotFoundError: No module named 'src'
```
컨테이너가 `Restarting (1)` 상태로 계속 재시작.

**원인**
`CMD ["python", "src/main.py"]`로 실행하면 Python이 `/app/src`를 sys.path에 추가.
`main.py` 내부에서 `from src.database import ...` 호출 시 `src`를 찾지 못함.

**해결**
모듈 모드로 실행 → Python이 `/app`을 sys.path에 추가:
```dockerfile
CMD ["python", "-m", "src.main"]
```

---

### 문제 5 — Cloudflare DNS 레코드 누락

**증상**
도메인 접속 불가 (`Could not resolve host`).

**원인**
`config.yml`에 ingress 항목만 추가하고 Cloudflare DNS CNAME 등록을 빠뜨림.

**해결**
```bash
cloudflared tunnel route dns <tunnel-id> lotto.nuclearbomb6518.com
```
→ Cloudflare DNS에 CNAME 자동 등록. cloudflared 서비스 재시작도 필요:
```bash
net stop cloudflared && net start cloudflared
```

---

## 최종 deploy.yml 흐름

```
1. Checkout & Build (GitHub Actions Ubuntu runner)
2. Setup SSH key (base64 decode)
3. Sync build output:
   a. pm2 stop profile-next
   b. rmdir next (기존 폴더 삭제)
   c. mkdir next, .next, .next/static, public
   d. tar | ssh (standalone, static, public, ecosystem.config.js)
4. Reload pm2:
   pm2 reload || pm2 start && pm2 save && schtasks /run /tn PM2-Resurrect
```

---

## GitHub Secrets 목록

| Secret | 값 |
|--------|-----|
| `DEPLOY_HOST` | `windows.nuclearbomb6518.com` |
| `DEPLOY_PORT` | `2222` |
| `DEPLOY_USER` | `ylswn` |
| `DEPLOY_PATH` | `C:\Users\ylswn\Projects\profile` |
| `DEPLOY_SSH_KEY` | deploy_windows 프라이빗 키 (base64 인코딩) |

---

## 무중단 배포 가능 여부

현재 방식은 **다운타임 있음** (pm2 stop → 파일 교체 → pm2 start).

Linux에서는 열린 파일 덮어쓰기 가능 → `pm2 reload`만으로 무중단 가능.
Windows에서 무중단 배포하려면 **Blue-Green 배포** 필요:
- `next-blue` / `next-green` 디렉토리 교대 사용
- 신규 버전을 유휴 디렉토리에 배포 후 pm2 cwd swap → reload
