# Cloudflare — 엣지 네트워크 · Zero Trust · Tunnel

> 2026-06-08 | Cloudflare, CDN, ZeroTrust, Tunnel, Workers, DNSSEC

## 한 줄 요약

Cloudflare는 도메인 DNS를 프록시해 모든 트래픽을 **전 세계 엣지(리버스 프록시/CDN)** 로 통과시키는 네트워크이며, 그 위에 보안(DDoS/WAF/SSL)·엣지 컴퓨트(Workers)·Zero Trust(Access/Tunnel)를 얹어 "origin 앞에 서는 글로벌 보안·성능 레이어"를 제공한다.

## 핵심 개념

### Cloudflare란 — DNS 프록시로 작동하는 엣지

```
방문자 → [ Cloudflare 엣지(전세계 PoP) ] → origin 서버
              ↑ 캐싱·보안·라우팅을 엣지에서 처리
```

동작 트릭 = **DNS**. 도메인을 Cloudflare로 옮기고 레코드를 **proxied(주황 구름)** 로 켜면 모든 요청이 엣지를 먼저 거친다 → **origin IP 은닉** + 엣지에서 캐싱/방어/SSL.

### 기능 카테고리

- **성능/CDN**: 엣지 캐싱(사용자 가까운 PoP에서 서빙), Argo 스마트 라우팅, 이미지 최적화, Brotli, HTTP/3(QUIC).
- **DNS**: authoritative DNS, DNSSEC(아래).
- **보안**: DDoS 방어, WAF(SQLi/XSS 룰), Bot 관리, Rate Limiting, SSL/TLS 모드.
- **엣지 컴퓨트**: Workers/Pages/R2/KV/D1/Durable Objects/Queues(아래).
- **Zero Trust(SASE)**: Access/Gateway/Tunnel/WARP(아래).

#### SSL/TLS 모드 (헷갈림 주의)

| 모드 | 방문자↔CF | CF↔origin |
|------|-----------|-----------|
| Flexible | HTTPS | **HTTP 평문** ⚠️ |
| Full | HTTPS | HTTPS(자체서명 허용) |
| **Full (strict)** | HTTPS | HTTPS(유효 인증서 검증) ✅ 권장 |

### Authoritative DNS & DNSSEC

DNS 계층: **Recursive Resolver**(1.1.1.1/ISP — 대신 질의·캐싱)와 **Authoritative DNS**(레코드를 보유한 "진실의 출처")가 있다. 도메인을 Cloudflare에 올리면 Cloudflare가 **authoritative DNS** 역할(빠른 anycast 응답).

**DNSSEC** = DNS 응답에 **전자서명**을 붙여 위조(캐시 포이즈닝)를 막는 것. Root → TLD → 도메인으로 이어지는 **신뢰 체인**으로 "이 응답은 진짜 권위 서버가 줬고 변조 안 됨"을 검증.

⚠️ DNSSEC은 **위변조 방지(인증/무결성)이지 암호화가 아니다.** 질의 내용을 암호화하는 건 DoH/DoT(별개).

| 기술 | 보호 | 종류 |
|------|------|------|
| DNSSEC | 응답 위조 방지 | 무결성/인증 |
| DoH/DoT | 질의 내용 암호화 | 기밀성 |

### 엣지 컴퓨트 — Workers vs Azure Functions

둘 다 서버리스지만 **실행 위치와 방식**이 다르다.

| | Azure Functions / Lambda | Cloudflare Workers(엣지) |
|---|---|---|
| 위치 | 특정 리전 | **전 세계 엣지**(사용자 근처) |
| 격리 | 컨테이너/마이크로VM | **V8 Isolate**(경량 샌드박스) |
| 콜드스타트 | 수백ms~수초 | **~5ms (거의 없음)** |
| 런타임 | 풀 런타임, 파일시스템 | JS/Wasm, 제약된 런타임 |
| 적합 | 무거운 백엔드 로직 | 요청 가공·인증·경량 API·전세계 저지연 |

핵심: Workers는 **V8 Isolate**(크롬 탭 격리 기술)로 한 프로세스에 수천 개를 격리 → 콜드스타트 거의 없이 **전 세계 엣지 어디서나** 실행. (컨테이너 vs isolate = 무거움 vs 경량 다중화)

### 엣지 데이터 — stateless가 아니라 "일관성 트레이드오프"

⚠️ 컴퓨트는 stateless라 막 뿌리지만, **데이터(stateful)는 "어디 게 진짜냐"가 생겨 아무 데나 못 둔다.** 그래서 제품마다 분산 전략이 다르다.

```
많이 뿌림(약한 일관성) ←──────────────→ 한 곳에 모음(강한 일관성)
KV(엣지캐시)  R2(리전)  D1(프라이머리+레플리카)  Durable Objects(단일위치)
```

| 제품 | 성격 | 유용한 예 |
|------|------|----------|
| **KV** | key-value용 CDN, **결과적 일관성**, 읽기 최적 | 피처플래그, URL 단축, API응답 캐시 |
| **R2** | S3 호환 스토리지 + CDN + **이그레스 0** | 이미지/영상 업로드, 다운로드 배포 |
| **D1** | SQLite 기반 가벼운 RDB(레플리카) | 소규모 앱 백엔드, 관계형 메타데이터 |
| **Durable Objects** | 단일 위치 + **강한 일관성** | 실시간 협업·채팅방, 정확한 카운터/재고, Rate Limiter |
| **Queues** | 비동기 작업 | 크롤링, 알림 발송, 미디어 후처리 |

⚠️ **KV는 Redis 대체가 아니다** — 결과적 일관성 + 쓰기 전파 지연. 빠른 쓰기/자주 바뀌는 유저 상태는 Durable Objects나 Redis. KV는 전 세계에 **같은 데이터를 캐시**(위치별 상태 X).

근본 이유 = **CAP 정리**: 네트워크 분할 환경에서 일관성과 가용성/지연을 동시에 완벽히 못 가짐. 강한 일관성을 원할수록 덜 퍼뜨린다(오히려 한 곳에 모음).

### Zero Trust — "안에 있다고 믿지 마라"

전통 보안 = **성곽과 해자**("사내망=신뢰"). 원격근무·클라우드로 경계가 사라지고, VPN 한 번 뚫리면 내부 전체 노출(측면 이동) → 붕괴.

Zero Trust 원칙: **"절대 신뢰하지 말고 항상 검증"** — 위치로 신뢰하지 않고 매 요청마다 신원·디바이스를 검증, 최소 권한.

| 제품 | 역할 |
|------|------|
| **Access** | 앱 앞단 인증 게이트(IdP 연동) — VPN 대체 |
| **Gateway** | 아웃바운드 보안(DNS/HTTP 필터링) — SWG |
| **Tunnel** | origin을 포트 개방 없이 엣지에 연결 |
| **WARP** | 디바이스 클라이언트(트래픽을 엣지로) |

조합: **Tunnel(안전 노출) + Access(인증 게이트)** = 내부 서비스를 포트 없이 + 로그인한 사람만.

### Cloudflare Tunnel

**문제**: 집/사내 서버 노출 = 포트포워딩 + 고정IP + 방화벽 개방 → IP/포트 노출, 위험.

**해결**: `cloudflared`가 **아웃바운드로만** 엣지에 영구 연결.
```
[방문자] → Cloudflare 엣지 ←─(아웃바운드 터널)─ [cloudflared(서버 안)] → localhost:서비스
```
→ **인바운드 포트 0개**, origin IP 은닉, 고정IP 불필요.

```yaml
# ~/.cloudflared/config.yml
tunnel: <uuid>
credentials-file: <uuid>.json
ingress:
  - hostname: app.example.com
    service: http://localhost:3000
  - service: http_status:404        # 매칭 안 되면 404 (필수 마지막 줄)
```
DNS는 **CNAME**: `app.example.com` → `<uuid>.cfargotunnel.com`.

| 방식 | 포트 개방 | 고정IP | 특징 |
|------|----------|--------|------|
| 포트포워딩 | 필요 ⚠️ | 필요 | IP/포트 노출 |
| ngrok | 불필요 | 불필요 | 임시(개발용) |
| VPN | (서버단) | 보통 필요 | 네트워크 전체 접근(과함) |
| **Cloudflare Tunnel** | **불필요** | 불필요 | 영구 + 앱 단위 + IP 은닉 |

## 핵심 질의응답

**Q. Authoritative DNS가 뭔가?**
A. 도메인 레코드를 실제로 보유하고 권위 있게 답하는 서버. Recursive Resolver(대신 질의·캐싱)와 구분된다. Cloudflare에 도메인을 올리면 Cloudflare가 그 역할을 한다.

**Q. DNSSEC = 암호화?**
A. 아니다. 응답에 서명을 붙여 위조를 막는 인증/무결성 기술이다. 질의 내용 암호화는 DoH/DoT로 별개.

**Q. 엣지 컴퓨트 = Azure Functions?**
A. 같은 서버리스지만 Workers는 리전이 아니라 전 세계 엣지에서 V8 isolate로 콜드스타트 거의 없이 돈다. 가볍고 저지연이나 런타임은 제약적.

**Q. DB/스토리지/큐를 엣지에서 돌리면 가벼운가?**
A. 컴퓨트의 "가벼움"과 다르다. 데이터는 상태가 있어 일관성에 따라 분산 전략을 달리한 것(KV는 뿌림/약한 일관성, Durable Objects는 모음/강한 일관성).

**Q. KV는 Redis인가?**
A. 아니다. 전 세계 복제 + 결과적 일관성 + 쓰기 지연 → key-value용 CDN(읽기 캐시)에 가깝다. 빠른 쓰기/유저 상태엔 Durable Objects나 Redis.

## 주의사항 / 자주 하는 실수

- **SSL Flexible 모드** — origin 구간 평문. Full (strict) 권장.
- **KV를 세션/실시간 상태에 사용** — 결과적 일관성으로 stale 가능. DO/Redis로.
- **DNSSEC을 암호화로 오해** — 위변조 방지일 뿐.
- **Tunnel에 ingress 마지막 catch-all 누락** — `http_status:404` 필수.

## 참고

- [Docker · 컨테이너](Docker-컨테이너.md) — 매니지드 컨테이너 배포 대안
- [인증 종합 (JWT/OAuth/OIDC)](../인증/인증-JWT-OAuth-OIDC.md) — Zero Trust의 AuthN/AuthZ
- [12/15-Factor App](../아키텍처/12-15-Factor-App.md) — Telemetry/Auth factor
- [Cloudflare Docs](https://developers.cloudflare.com/)
