# 인증 종합 — 세션 / JWT / OAuth / OIDC / Passport

> 2026-06-04 | 인증, 인가, JWT, 세션, OAuth2, OIDC, PKCE, Passport, CSRF, mTLS

## 한 줄 요약

인증의 모든 갈림길은 **"상태를 어디 두나(stateful/stateless), 누가 검증하나(자원서버/인가서버), 무엇을 증명하나(권한/신원)"**로 정리된다. 세션(stateful)은 제어가 쉽고 토큰/JWT(stateless)는 확장이 쉬운 정반대 트레이드오프. OAuth는 인가(권한 위임), OIDC는 그 위에 얹은 인증(신원). 모든 보안 조치는 결국 "노출·위조·탈취를 어떻게 막나"의 트레이드오프다.

> 어제/그제 학습과 연결 → [NestJS 아키텍처](../백엔드/NestJS-아키텍처.md)(Guard·생명주기), [FastAPI 기본](../백엔드/FastAPI-기본.md)(Depends 인증)

## 핵심 개념

### 1. 인증(authN) vs 인가(authZ)

| | Authentication | Authorization |
|---|---|---|
| 질문 | **"너 누구냐?"**(신원) | **"권한 있냐?"**(권한) |
| 순서 | **먼저** | 나중 |
| 예 | 로그인 | `@Roles('admin')` |

**401 vs 403 (이름이 헷갈림):**

| 코드 | 이름 | 진짜 의미 |
|------|------|----------|
| **401** | Unauthorized | 실제론 **"인증 실패"**(누군지 모름) → 로그인하면 풀림 |
| **403** | Forbidden | **인가 거부**(누군진 알지만 권한 없음) → 로그인해도 막힘 |
| **404** | Not Found | **존재 자체를 숨김**(민감 리소스) — GitHub 비공개 repo 방식 |

- 401: 토큰 없음/만료 → "로그인부터"
- 403: 일반 유저가 `/admin` → "넌 안 돼"
- 404 위장: 403을 주면 "뭔가 있다"가 노출되므로, 숨겨야 하면 **일관되게** 404
- 인가 모델: **RBAC**(역할, 가장 흔함·`@Roles`), **ABAC**(속성·"본인 글만"), **ACL**(리소스별)

### 2. 세션 기반 vs 토큰 기반 — stateful vs stateless

| | 세션(Stateful) | 토큰/JWT(Stateless) |
|---|---|---|
| 상태 위치 | **서버**(저장소) | **토큰**(클라이언트) |
| 요청당 | 저장소 조회 | 서명 검증만 |
| 즉시 무효화 | **쉬움**(삭제) | **어려움**(만료까지 유효) |
| 수평 확장 | 어려움(세션 공유 필요) | **쉬움**(어디서든 검증) |
| 적합 | 모놀리식 | **MSA·분산·서버리스** |

> **정반대 트레이드오프: 세션 = 제어 쉬움/확장 어려움, 토큰 = 확장 쉬움/제어 어려움.**

어제 "Node 프로세스 다중화"와 직결: 세션 기반은 N개 프로세스가 세션 공유(Redis sticky) 필요, 토큰은 stateless라 어느 프로세스든 서명만 검증 → 분산에서 JWT 선호.

**JWT 약점 보완:** 발급 후 못 막음 → 짧은 만료 + refresh + 블랙리스트. 단 블랙리스트를 붙이면 다시 stateful로 회귀(모순).

### ★ REST stateless 정정 (중요)

> **"Redis에 세션 두면 stateless"는 틀렸다.** REST의 stateless = "서버가 세션 상태를 보관하지 않고 각 요청이 자기완결적". Redis는 상태의 **저장 위치만** 옮긴 것이고, 서버가 세션 상태를 유지하므로 **여전히 stateful**이다.

```
세션ID + Redis : 클라는 "보관증(세션ID)"만, 짐(상태)은 서버측 창고(Redis) → STATEFUL
JWT            : 클라가 "짐 전체(서명된 상태)"를 들고 다님, 서버는 검증만 → STATELESS
```

- **"REST stateless"**(아키텍처 제약) ≠ **"stateless 애플리케이션 서버"**(인스턴스가 로컬 상태 없음 = 인프라/배포 용어). Redis 외부화로 얻는 건 후자.
- **확장성 ≠ stateless**: 세션+Redis도 확장은 됨(공유 저장소). 확장성은 "상태 공유"로, REST stateless는 "상태를 서버에 안 둠(JWT)"으로 얻는 별개 축.

**그럼 세션 쓰면 REST API 아닌가?** 엄밀히는 stateless 제약 위반이라 완전한 RESTful은 아니다. 하지만 REST는 6제약의 **스펙트럼**이고(Richardson 성숙도 0~3, 대부분 Level 2에 머묾, HATEOAS까지 가는 곳 극소수), 업계에선 세션을 써도 REST API라 부른다. 정합적으로 가려면 stateless인 JWT.

### 3. JWT 구조 — `header.payload.signature`

```
eyJhbGci... . eyJzdWIi... . dBjftJeZ...
  Header        Payload       Signature   (각각 base64url, '.'로 연결)
```

- **Header**: `{ "alg": "HS256", "typ": "JWT" }`
- **Payload**: claims — `iss`(발급자) `sub`(주체) `aud`(대상) `exp`(만료) `iat`(발급) `jti`(고유ID)
- **Signature**: `HMACSHA256(base64(header)+"."+base64(payload), secret)`

**★ 핵심 1 — base64url은 암호화가 아니라 인코딩.** 누구나 디코딩해서 읽힘 → **payload에 민감정보 금지**.

**★ 핵심 2 — 왜 위변조 불가:** payload를 고쳐도 secret 없는 공격자는 올바른 서명을 못 만든다. 서버가 받은 header+payload로 서명을 다시 계산해 비교 → 불일치면 거부. **기밀성이 아니라 무결성·진정성을 보장**(그래서 서버 저장 없이 stateless 신뢰 가능).

**HS256 vs RS256:**

| | HS256(HMAC, 대칭) | RS256(RSA, 비대칭) |
|---|---|---|
| 키 | 하나의 secret로 서명+검증 | private 서명 / public 검증 |
| 적합 | 단일 서버 | **MSA·제3자 검증**(검증자에 public만) |
| 쓰는 곳 | 자체 인증 | **OAuth/OIDC** |

**취약점(면접 단골):**
- `alg: none` — 서명 없이 통과 시도 → none 거부
- alg confusion(RS256→HS256) — public key를 HMAC secret으로 악용 → **검증 시 알고리즘 화이트리스트 고정**

```javascript
jwt.verify(token, key, { algorithms: ['RS256'] });  // 토큰의 alg를 믿지 말고 고정
```

### 4. Access token vs Refresh token

"짧으면 안전하지만 자주 재로그인, 길면 편하지만 탈취 위험" 모순을 역할 분리로 해결:

| | Access | Refresh |
|---|---|---|
| 수명 | 짧음(5~15분) | 김(며칠~주) |
| 용도 | API 요청마다 | **재발급에만** |
| 저장(서버) | 안 함(stateless) | **함**(DB/Redis, stateful) |

```
로그인 → access(15m)+refresh(7d) → access로 호출 → 만료 401
   → refresh로 재발급(새 access+refresh) → refresh 만료 시 재로그인
```

> **2번 트레이드오프의 실무 해법:** access는 stateless(빠름·분산), refresh는 stateful(무효화·로그아웃 제어). 로그아웃 = 서버에서 refresh 삭제 → access는 최대 15분 뒤 소멸.

**Rotation(회전):** refresh 쓸 때마다 새로 발급+이전 폐기(1회용). 폐기된 refresh 재사용 감지 → 도난 의심 → 전체 무효화. refresh는 **HttpOnly 쿠키 + 서버 저장**.

### 5. OAuth 2.0 — 위임 인가

> **비밀번호를 제3자 앱에 주지 않고 "특정 권한만" 위임하는 인가(authorization) 프로토콜. 인증이 아니다.** "OAuth로 로그인"은 부정확 — 인증은 OIDC가 담당.

**4역할:** Resource Owner(사용자) · Client(내 앱) · Authorization Server(토큰 발급, 구글 등) · Resource Server(자원 API)

**Authorization Code Flow:**
```
사용자 "구글 연동" → 인가서버 리다이렉트(client_id,scope,state)
→ 구글에서 로그인+동의(비번은 구글에만) → authorization code(일회용) 받음
→ 백엔드가 code+client_secret으로 토큰 교환(URL 노출 방지) → access token으로 자원 접근
```

**PKCE — secret을 못 숨기는 SPA·모바일용:**
```
1. verifier 생성(클라 메모리에만) → challenge = hash(verifier)
2. challenge(해시)만 인가요청에 첨부 (인가서버가 저장)
3. authorization code 발급 ← 여기서 공격자가 code 가로챌 수 있음
4. 토큰교환 때 verifier 원본 전송 → 인가서버가 hash(verifier)==challenge 검증
5. code만 털려도 verifier 없으면 토큰 못 받음
```
- **암호화가 아니라 일회용 증명.** client_secret과 다름(secret 없는 환경의 대체, 매번 생성하는 임시값)
- 위협 모델 = **authorization code 가로채기**(클라 전체 장악은 범위 밖)
- challenge를 해시로 보내 역산 방지. **검증은 인가서버가, 토큰 발급 전에**
- 대부분 OAuth 라이브러리에 **자동 내장**(직접 구현 불필요)
- `state`=CSRF 방어, `scope`=권한 범위

> 인가서버는 challenge/code/refresh/세션을 저장하므로 **stateful**. "JWT stateless"는 access token을 검증만 하는 **자원서버** 관점. 시스템의 상태는 인가서버에 모인다.

### 6. OIDC — 소셜 로그인의 정체

**OAuth만으론 인증이 위험한 이유(token substitution):** access token은 "신원"·"내 앱 대상 여부"를 보장 안 함. 공격자가 다른 앱용 access token을 내 앱에 던지면 → 내 앱이 그걸로 프로필 조회 → 남의 계정 탈취.

**OIDC = OAuth 위에 얹은 인증 레이어. 산출물 = ID Token(항상 JWT):**

| | Access Token | ID Token |
|---|---|---|
| 목적 | 자원 접근(권한) | 신원 증명(누구) |
| 소비자 | Resource Server | **Client(내 앱)** |
| 핵심 | scope | **`aud`=내 client_id**, sub, 이메일 |

> ID token의 `aud`에 **내 client_id**가 박혀 "이 토큰이 내 앱을 위한 것"을 검증 → **token substitution 방어**. `nonce`로 replay 방어. 서명 RS256.

- **소셜 로그인** = scope에 `openid` 추가 → 응답에 id_token → 검증해서 신원 확정
- 검증: 서명(구글 public key) → `iss` → **`aud`(내 client_id)** → `exp` → `nonce`
- **사용처: ID token은 로그인 시점 신원 확인용, 이후 API 요청은 access token으로.**

### 7. Passport.js — 인증 전략 추상화

> 인증 방식을 **Strategy(전략)**로 모듈화: `passport-local`(세션), `passport-jwt`(JWT), `passport-google-oauth20`(OAuth/OIDC). 전략을 갈아끼워도 컨트롤러는 그대로.

```typescript
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({ jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
            secretOrKey: SECRET, algorithms: ['HS256'] });  // 알고리즘 고정
  }
  async validate(payload) { return { userId: payload.sub }; }  // → req.user
}

@UseGuards(AuthGuard('jwt'))   // 어제 Guard 생명주기에 끼움
@Get('profile')
getProfile(@Req() req) { return req.user; }  // validate() 반환값
```

> **Passport="어떻게 검증", AuthGuard="언제 적용".** 어제 "Guard가 `req.user`를 채운다"의 그 값이 Strategy `validate()` 반환값. `Spring Security ≈ Passport + AuthGuard`. 세션 전략은 serialize/deserialize 추가(JWT는 불필요).

### 8. 쿠키 보안 + CSRF

```
Set-Cookie: token=xxx; HttpOnly; Secure; SameSite=Lax
```

| 플래그 | 막는 것 |
|--------|---------|
| **HttpOnly** | XSS 토큰 탈취(JS 접근 불가) |
| **Secure** | 네트워크 도청(HTTPS만) |
| **SameSite** | CSRF(크로스사이트 전송 제어): Strict/Lax(기본)/None |

**CSRF:** 브라우저가 쿠키를 **자동 첨부**하는 걸 악용(로그인 상태에서 악성 사이트가 요청 위조). 방어 = **SameSite**(1차) + **CSRF 토큰**.

**XSS vs CSRF 트레이드오프:**

| 저장 | XSS | CSRF |
|------|-----|------|
| HttpOnly 쿠키 | 안전 | 주의(SameSite로) |
| localStorage+헤더 | **취약** | 안전 |

→ 권장: **HttpOnly+Secure+SameSite 쿠키 + (민감작업) CSRF 토큰**, XSS는 CSP·입력검증 별도.

### 9. mTLS — 상호 TLS

| | 일반 TLS | mTLS |
|---|---------|------|
| 인증 방향 | 서버만 | **양방향**(클라도 인증서) |
| 용도 | 웹 | **서비스 간**(MSA), zero-trust |

> 주로 **기계/서비스 인증**(사람 로그인 아님). MSA 서비스 간 통신을 서비스 메시(Istio)가 자동 mTLS로. 인증서 관리(발급·회전·폐기)가 부담. 사용자 인증은 JWT/OAuth, 서비스 간은 mTLS.

---

## 핵심 질의응답

**Q. payload를 고쳐서 `role: admin`으로 보내면 admin이 되나?**
A. 안 된다. payload를 고치면 서명이 생성 때와 달라지고, secret 없는 공격자는 올바른 서명을 못 만들어 검증에서 거부된다.

**Q. "Redis에 세션 두면 stateless"가 맞나?**
A. 틀렸다(과거 부정확한 설명 정정). REST stateless는 "서버가 세션 상태를 안 둠". Redis는 저장 위치만 옮긴 것이라 여전히 stateful. 진짜 stateless는 상태를 토큰에 담는 JWT. "stateless 앱 인스턴스"(인프라 용어)와 "REST stateless"는 다르고, 확장성과 stateless도 별개 축.

**Q. 세션을 쓰면 REST API가 아닌가?**
A. 엄밀히는 stateless 제약 위반이라 완전한 RESTful은 아니다. 단 REST는 스펙트럼이고 대부분 API가 Level 2의 느슨한 REST라 업계에선 그래도 REST API라 부른다. 정합적이려면 JWT.

**Q. PKCE가 왜 필요한가? client_secret과 다른가?**
A. client_secret을 못 숨기는 SPA·모바일용. 영구 secret 대신 매번 생성하는 일회용 verifier로 "처음 인가 요청한 클라가 맞다"를 증명. 암호화가 아니라 일회용 증명. authorization code 가로채기를 막는다(code만 털려도 verifier 없으면 토큰 못 받음).

**Q. PKCE를 검증하는 건 내 백엔드인가?**
A. 아니다. 토큰을 발급하는 Authorization Server(구글 등 외부 OAuth 서버, 또는 자체 운영 auth 서버)가 토큰 발급 전에 검증한다. 비즈니스 API를 처리하는 Resource Server(앱 백엔드)는 관여 안 함.

**Q. 인가서버는 stateless인가?**
A. 아니다. challenge·code·refresh·세션을 저장하므로 stateful. "JWT stateless"는 access token을 검증만 하는 자원서버 관점이고, 상태는 인가서버에 모인다.

**Q. OAuth access token으로 프로필 조회해서 로그인하면 안 되나?**
A. 위험하다(token substitution). access token은 "내 앱을 위해 발급됐는지"를 보장 안 해서, 공격자가 다른 앱용 토큰을 던지면 남의 계정으로 로그인될 수 있다. OIDC의 ID token은 `aud`에 내 client_id가 박혀 이를 방어한다.

**Q. ID token과 access token, 매 API 요청엔 뭘 쓰나?**
A. access token. ID token은 로그인 시점 신원 확인용(audience가 내 앱). access token이 자원 접근용.

**Q. HttpOnly 쿠키는 무슨 공격을 막나?**
A. XSS로 `document.cookie`를 통해 토큰을 탈취하는 것. 단 쿠키는 CSRF에 주의해야 하므로 SameSite를 함께 쓴다.

## 주의사항 / 자주 하는 실수

- **JWT payload에 민감정보** — 인코딩일 뿐 누구나 읽음
- **알고리즘 미고정** — `alg:none`·confusion 취약 → `algorithms` 화이트리스트
- **JWT를 localStorage에** — XSS 취약 → HttpOnly 쿠키 권장
- **블랙리스트로 JWT 무효화** — stateful 회귀(트레이드오프 인지)
- **access token을 인증에 사용**(OAuth만으로 로그인) — token substitution → OIDC ID token 사용
- **ID token을 매 API에 전송** — 용도 오용(access token이 맞음)
- **쿠키에 SameSite 미설정** — CSRF 노출
- **401/403/404 일관성 없음** — 그 차이로 리소스 존재가 추론됨

## 참고

- [JWT 공식](https://jwt.io/) · [OAuth 2.0](https://oauth.net/2/) · [OIDC](https://openid.net/connect/)
- [NestJS 아키텍처](../백엔드/NestJS-아키텍처.md) ← Guard/요청 생명주기, AuthGuard
- [FastAPI 기본](../백엔드/FastAPI-기본.md) ← Depends 인증, 전역 인증+공개 제외
- (예정) 모던 API 아키텍처 1·2장 — REST 기초/리소스 모델링
