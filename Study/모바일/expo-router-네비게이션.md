# expo-router — 파일 기반 네비게이션

> 2026-06-10 | expo-router, React Navigation, 라우팅, Expo, 딥링크

## 한 줄 요약

expo-router는 **`app/` 폴더의 파일 구조 자체를 네비게이션 트리로 삼는** 규약 레이어로, React Navigation 위에 Next.js App Router 스타일의 선언적 라우팅을 씌운 것이다 — "파일 경로 = URL = 네비게이션 상태"를 하나로 통일한다.

> 같은 셸의 OTA/Hermes/Bridge/푸시는 → [RN+Expo WebView 래퍼 아키텍처](RN-Expo-WebView래퍼-아키텍처.md). 프레임워크 선택은 → [RN vs Flutter 비교](RN-vs-Flutter-비교.md)

## 핵심 개념

### 왜 파일 기반 라우팅인가

이전 표준인 **React Navigation**은 명령형 + 중앙 집중 설정이었다:

```tsx
const Stack = createNativeStackNavigator();
<Stack.Navigator>
  <Stack.Screen name="Home" component={HomeScreen} />
  <Stack.Screen name="Post" component={PostScreen} />
  {/* 화면 추가마다 import + 등록 두 곳을 손봐야 함 */}
</Stack.Navigator>
```

통증: 중앙 등록부가 비대해지고, 라우트 구조가 코드에 묻히고, 딥링크(URL↔화면) 매핑을 `linking` 설정에 **수동으로 또** 작성해야 했다.

expo-router의 답 — **파일 구조가 곧 라우트 트리**:

```
app/
├── index.tsx       →  /
├── profile.tsx     →  /profile
├── post/[id].tsx   →  /post/:id   (동적)
└── _layout.tsx     →  이 그룹의 네비게이터
```

파일을 만들면 그게 라우트. "어떤 화면이 있나"는 `app/` 폴더만 보면 끝. URL이 파일 경로에서 자동 생성 → **딥링크가 공짜**.

### 파일 경로 = URL = 네비게이션 상태 (패러다임의 본질)

```
app/post/[id].tsx  →  /post/42  →  Post 화면(id=42)이 스택에 푸시
```

기존엔 이 셋을 따로 관리(화면 등록 / linking 설정 / navigate 호출)했는데, expo-router는 **파일 경로 하나로 셋을 동기화**한다. 웹의 URL 멘탈 모델을 네이티브로 가져온 것.

### Next.js App Router와의 평행

[Next App Router](../프론트엔드/Next.js-기본.md)를 안다면 거의 그대로 옮겨진다 — 의도된 설계.

| 개념 | Next.js | expo-router |
|------|---------|-------------|
| 레이아웃 | `layout.tsx` | `_layout.tsx` |
| 동적 라우트 | `[id]/page.tsx` | `[id].tsx` |
| catch-all | `[...slug]` | `[...rest]` |
| 라우트 그룹 | `(group)` | `(group)` |
| 링크 | `<Link href>` | `<Link href>` |

→ **"네이티브판 Next.js App Router"**. 단 동작은 다르다(아래).

### expo-router는 React Navigation 위에 산다 — Next와 갈리는 핵심

**웹은 "페이지 교체", 네이티브는 "스택에 쌓기"**:

```
[Next 웹]  /a → /b   화면 A를 버리고 B로 교체. 뒤로가기 = 브라우저 히스토리
[expo-router]  /a → /b   A 위에 B를 쌓음. A는 메모리에 살아있음
  ┌─────────┐
  │ 화면 B   │ ← 위로 슬라이드 등장
  ├─────────┤
  │ 화면 A   │ ← 살아있음! 뒤로가기하면 스크롤·입력 상태 그대로 복원
  └─────────┘
```

화면 A가 안 죽으므로 스크롤 위치·입력값이 보존된다. 그래서 Next엔 없는 개념 — **"어떻게 쌓을 것인가"(네비게이터)**가 필요하다.

추상화 계층:

```
expo-router            ← 파일 규칙 → 라우트 트리 자동 생성 (Next 스타일 껍데기)
  └ React Navigation   ← 실제 네비게이션 엔진 (스택/탭/제스처/헤더/상태)
      └ react-native-screens  ← 네이티브 화면 (iOS UINavigationController / Android Fragment)
```

⚠️ expo-router는 라우팅 엔진이 아니라 **React Navigation 위에 씌운 파일 규약 레이어**. 세밀한 옵션(헤더 커스텀, 제스처, 전환 애니메이션)은 **아래 React Navigation의 `options`로 내려가서** 해결한다 — "expo-router를 배운다 = React Navigation도 안다".

⚠️ **서버가 (거의) 없다**: Next App Router의 핵심인 Server Components/SSR/서버 액션은 네이티브 앱엔 해당 없음. expo-router 화면은 사실상 전부 클라이언트 컴포넌트(`"use client"` 불필요).

## 파일 시스템 규칙

### `app/` 디렉토리 매핑

| 패턴 | 의미 | 예 |
|------|------|----|
| `index` | 폴더 기본 라우트 | `app/index.tsx` → `/` |
| `[param]` | 단일 동적 세그먼트 | `[id]` → `/post/42` |
| `[...rest]` | catch-all | `[...rest]` → `/a/b/c` |
| `_파일` | 라우트 제외(레이아웃) | `_layout.tsx` |
| `(group)` | 라우트 그룹(URL 제외) | `(tabs)` |

⚠️ Next은 `app/post/[id]/page.tsx`(폴더+page)인데, expo-router는 **파일 자체가 라우트**(`app/post/[id].tsx`) — `page.tsx` 같은 고정 파일명 없음.

해석 우선순위: **정적 > 동적 > catch-all** (`/post/new`는 `new.tsx`가 `[id].tsx`보다 우선).

### `+` 접두사 특수 파일 (expo-router 고유)

```
app/+not-found.tsx     → 404 (매칭 실패 fallback, 잘못된 딥링크 안전망)
app/+html.tsx          → 웹 빌드 HTML 셸 (네이티브 무시)
app/+native-intent.tsx → 외부 딥링크 URL → 라우트 변환 훅
```

### `_layout.tsx` & 네비게이터

| 네비게이터 | UX | 화면 생명주기 | 사용처 |
|-----------|-----|-------------|--------|
| **Stack** | 위로 쌓임, 슬라이드 | 뒤 화면 살아있음 | 목록→상세 |
| **Tabs** | 하단 탭 바 | 탭들 동시 생존 | 앱 최상위 섹션 |
| **Drawer** | 옆 서랍 | 메뉴 항목 생존 | 사이드 메뉴 앱 |

```tsx
// Stack
<Stack>
  <Stack.Screen name="index" options={{ title: '홈' }} />
  <Stack.Screen name="post/[id]" options={{ title: '게시물' }} />
</Stack>

// Tabs — 같은 파일들을 하단 탭으로
<Tabs screenOptions={{ headerShown: false }}>
  <Tabs.Screen name="index" options={{ title: '홈' }} />
  <Tabs.Screen name="feed"  options={{ title: '피드' }} />
</Tabs>
```

- `screenOptions`(네비게이터 전체) vs `options`(화면별 오버라이드). 이 옵션들은 사실 **React Navigation 옵션**.
- 데이터 의존 옵션(게시물 제목을 헤더에)은 화면 내부에서 `<Stack.Screen options={{ title }} />`로 동적 지정.

### 중첩 네비게이터 (실전 핵심)

루트 Stack 안에 Tabs 중첩 — 가장 흔한 구조:

```
app/
├── _layout.tsx           → 루트 Stack
├── (tabs)/
│   ├── _layout.tsx       → Tabs (홈/피드/설정)
│   └── index.tsx, feed.tsx, settings.tsx
└── post/[id].tsx         → 상세 (탭 위로 풀스크린 푸시)
```

```tsx
// app/_layout.tsx
<Stack>
  <Stack.Screen name="(tabs)" options={{ headerShown: false }} />  {/* ⚠️ */}
  <Stack.Screen name="post/[id]" options={{ title: '게시물' }} />
</Stack>
```

탭은 나란히, 게시물 상세는 탭 위로 풀스크린으로 덮으며 등장(탭 바 가림). 바깥 Stack이 상세를 푸시 → 뒤로가기 시 탭 복귀.

⚠️ **이중 헤더 함정**: 중첩하면 바깥 Stack 헤더 + 안쪽 Tabs 헤더가 둘 다 그려진다. 바깥에서 `headerShown: false`로 한쪽을 꺼야 함. expo-router 중첩 최다 실수.

### Route Groups `(group)`

URL에 안 드러나는 폴더(Next과 동일). expo-router에선 **그룹마다 다른 네비게이터**를 거는 게 핵심:

```
app/(auth)/_layout.tsx   → 인증용 Stack (탭 없음)
app/(app)/_layout.tsx    → 메인 Tabs
```

로그인 상태에 따라 `(auth)` ↔ `(app)`를 통째로 전환(인증 플로우 뼈대).

⚠️ **공유 그룹 `(a,b)` 문법** (Next엔 없음): `app/(stack,modal)/detail.tsx` → 같은 화면을 `(stack)`에선 카드, `(modal)`에선 모달로. 코드 중복 없이 presentation만 다르게.

## 네비게이션 API

### 이동하기

```tsx
// 선언적
<Link href="/post/42">게시물</Link>
<Link href="/post/42" asChild><Pressable>...</Pressable></Link>  // 커스텀 버튼

// 명령형
const router = useRouter();
router.push('/post/42');     // 스택에 쌓기
router.replace('/home');     // 현재 교체 (뒤로가기 불가)
router.back();               // 스택 pop
router.dismiss();            // 모달 닫기

// 렌더 시점 리다이렉트
if (!user) return <Redirect href="/login" />;
```

⚠️ `asChild`: RN엔 `<a>`가 없으니 아무 컴포넌트나 링크로 만드는 핵심 패턴.

### push vs replace vs navigate ⚠️ 모바일 스택 동작

```
현재: [홈] → [피드]
push('/profile')    → [홈] → [피드] → [프로필]   (무조건 쌓음, 중복 가능)
replace('/profile') → [홈] → [프로필]            (교체, 뒤로가기 불가)
navigate('/feed')   → [홈] → [피드]              (이미 있으면 점프, 안 쌓음)
```

⚠️ **로그인 성공 후 `push('/home')` ❌** → 뒤로가기로 로그인 화면 복귀. 반드시 `replace('/home')`.

### 동적 params

```tsx
// app/post/[id].tsx  →  /post/42?ref=home
const { id, ref } = useLocalSearchParams();   // { id: "42", ref: "home" }
// catch-all → 배열: const { rest } = useLocalSearchParams();  // ["a","b","c"]
```

⚠️ **함정 1 — 모든 param은 string**: `id + 1` → `"421"`(문자열 결합). `Number(id)` 필수.

⚠️ **함정 2 — Local vs Global**:
```
useLocalSearchParams()   → "이 화면"에 고정된 params (기본, 권장)
useGlobalSearchParams()  → 현재 활성 URL params (비활성 화면까지 리렌더 → 성능 주의)
```
스택 `[/post/1] → [/post/2]`에서 `/post/1` 화면: local은 `{id:"1"}` 고정, global은 `{id:"2"}`(활성 URL).

⚠️ 큰 데이터를 params로 넘기지 말 것 — id만 넘기고 상세에서 다시 fetch하거나 [TanStack Query](../상태관리/서버상태-라이브러리비교.md) 캐시 공유. params엔 식별자만.

### Typed Routes

```tsx
router.push('/post/42');   // ✅
router.push('/pots/42');   // ❌ 타입 에러 (오타 컴파일 타임 차단)
```

```json
// app.json — { "expo": { "experiments": { "typedRoutes": true } } }
```

`app/` 스캔으로 라우트 타입 자동 생성(`.expo/types/`). 라우트는 문자열이라 리팩터링에 취약한데, 파일을 옮기면 참조처가 즉시 타입 에러 → 안전한 일괄 수정. 켜두면 손해 없음.

## 플랫폼 / 실전

### 플랫폼 간 네비게이션 차이 (Next엔 없는 영역)

같은 `app/` 코드가 플랫폼별 "뒤로가기"로 매핑됨:

```
iOS      → 왼쪽 엣지 스와이프, 헤더 "< 뒤로"
Android  → 하드웨어/제스처 백버튼 (시스템 레벨)
웹       → 브라우저 뒤로가기, URL 바
```

- **iOS**: 엣지 스와이프 기본. 결제/작성 중엔 `gestureEnabled: false`. 모달은 아래로 스와이프해서 닫음.
- **Android**: 백버튼 가로채기 — "작성 중 나갈까요?" 경고:
  ```tsx
  useFocusEffect(() => {
    const sub = BackHandler.addEventListener('hardwareBackPress', () => {
      if (hasUnsavedChanges) { showExitConfirm(); return true; }  // true=막음
      return false;
    });
    return () => sub.remove();
  });
  ```
- **웹**: 각 화면이 진짜 URL을 가져 새로고침/북마크/뒤로가기 작동. 단 스택 개념이 약해(웹은 교체) "뒤 화면 살아있음" 가정 코드는 다르게 동작.
- **공통**: 콘텐츠는 `SafeAreaView`로 감싸 노치·홈 인디케이터 회피(헤더/탭은 자동).

### 인증 플로우 패턴

**방식 1 — `<Redirect>` 가드 (모든 버전)**:
```tsx
// app/(app)/_layout.tsx — 보호 그룹 전체 가드
if (isLoading) return <SplashScreen />;            // ⚠️ 세션 복원 대기
if (!session) return <Redirect href="/sign-in" />;
return <Stack />;
```
⚠️ `isLoading` 필수 — 시작 시 SecureStore에서 토큰 읽는 동안 session이 잠깐 null → 로그인 유저도 로그인 화면 깜빡임.

**방식 2 — `Stack.Protected guard` (v5/SDK 53+)**:
```tsx
// app/_layout.tsx
<Stack>
  <Stack.Protected guard={!!session}>
    <Stack.Screen name="(app)" />
  </Stack.Protected>
  <Stack.Protected guard={!session}>
    <Stack.Screen name="sign-in" />
  </Stack.Protected>
</Stack>
```
guard가 false인 화면은 라우터에서 제외 → 자동 리다이렉트. `session`이 바뀌면 보이는 화면 집합이 자동 전환(별도 `replace` 불필요).

⚠️ 세션 저장은 `expo-secure-store`(iOS Keychain/Android Keystore). `AsyncStorage` 평문 저장 ❌.

### 모달 & presentation

```tsx
<Stack.Screen name="modal" options={{ presentation: 'modal' }} />
```

| presentation | 동작 |
|--------------|------|
| `card`(기본) | 옆에서 슬라이드 |
| `modal` | 아래에서 위로 |
| `transparentModal` | 배경 비치는 팝업 |
| `formSheet` | iOS 폼 시트(부분 높이) |
| `fullScreenModal` | 전체 화면 |

닫기는 `router.dismiss()`/`dismissAll()`(back과 구분). iOS는 아래로 스와이프해서 닫힘.

### Deep Linking & Universal

파일 경로가 이미 URL이라 **파일 생성 = 딥링크 가능**. 별도 `linking` 설정 불필요.

```
1. Custom scheme   myapp://post/42        app.json "scheme" 한 줄
2. Universal Links (iOS) / App Links (Android)  https://myapp.com/post/42
   → 설치 시 앱으로, 미설치 시 웹 폴백. 도메인 증명 파일 필요
     (iOS apple-app-site-association, Android assetlinks.json)
```

⚠️ 같은 `app/` 코드가 iOS·Android·웹 빌드 → **하나의 URL이 모든 플랫폼에서 같은 화면**(universal). 복잡한 URL 변환은 `+native-intent.tsx`의 `redirectSystemPath`.

## Next.js 대응 매핑 (웹앱 팀 관점)

라우팅 코어는 **Next가 원조** — expo-router가 Next를 모바일로 가져온 것.

| expo-router | Next.js 대응 | 비고 |
|-------------|-------------|------|
| 파일 기반 라우팅 / `_layout` | App Router / `layout.tsx` | Next가 먼저 |
| `[id]`, `useLocalSearchParams` | `[id]`, `useParams`/`useSearchParams` | 동일 |
| `<Link>`, `useRouter` | `<Link>`, `useRouter` | 거의 동일 |
| Typed Routes | `experimental.typedRoutes` | 둘 다 있음 |
| `<Redirect>` / 인증 가드 | **Middleware + `redirect()`** | ⭐ Next가 서버사이드라 더 안전 |
| `presentation:'modal'` | **Parallel + Intercepting Routes** | ⭐ Next도 되지만 복잡 |
| 딥링크 / Universal | (불필요 — 웹은 URL이 본질) | |
| Stack/Tabs/Drawer, 제스처, 스택 생명주기 | ❌ 없음 | 모바일 전용 |

- **인증**: expo-router는 클라 가드(앱이 화면 그리기 직전). Next는 **middleware로 요청 단계 차단** → 보호 콘텐츠가 클라에 안 내려감(더 안전). 모바일은 코드가 기기에 다 있어 서버 가드 불가.
- **모달**: Next은 Parallel Routes(`@modal`) + Intercepting Routes(`(.)photo`)로 "URL 가진 모달". 더 강력하지만 개념이 어려움.

→ 웹앱 팀에겐 expo-router = **"아는 Next를 네이티브로 확장하는 도구"**. 정작 깊게 팔 건 Next의 Middleware/Parallel/Intercepting Routes.

## 핵심 질의응답

**Q. Next랑 문법이 똑같으면 뭘 더 배워야 하나?**
A. 표면 문법은 거의 같다. 학습 가치는 "Next와 다른 20%" — 웹은 페이지 교체, 네이티브는 스택에 쌓기(뒤 화면 생존). 그래서 Next엔 없는 네비게이터(Stack/Tabs/Drawer), 제스처, 모달 presentation, 딥링크가 핵심.

**Q. 라우트 그룹도 Next랑 같나?**
A. URL에서 폴더명이 빠지는 핵심은 동일. 단 expo-router는 "그룹마다 다른 네비게이터"를 거는 용도(인증 `(auth)` vs 메인 `(app)`)로 더 적극 쓰고, `(a,b)` 공유 그룹 문법은 Next에 없다.

**Q. 인증은 페이지별로 묶나, 미들웨어로 처리하나?**
A. 모바일은 서버가 없어 클라 가드(`<Redirect>` 또는 `Stack.Protected`)로 보호 그룹을 묶는다. Next 웹은 미들웨어/서버 `redirect()`로 요청 단계에 한 번 거르는 게 더 중앙집중적·안전. 방향이 다르다.

**Q. 회사가 웹앱(Next)인데 expo-router 비슷한 게 웹에도 있나?**
A. 반대로 Next가 원조다. 라우팅 코어는 동등하고, 인증 가드는 Next가 서버사이드라 더 안전, 모달은 Next가 Parallel+Intercepting Routes로 더 강력하지만 복잡. 네이티브 스택/제스처/딥링크만 모바일 전용.

**Q. params로 객체를 넘겨도 되나?**
A. 안 된다. params는 URL에 실리는 string이라 큰 객체를 JSON으로 넣으면 URL 비대 + 직렬화 비용. id만 넘기고 상세에서 fetch하거나 전역 상태/쿼리 캐시로 공유.

**Q. WebView 래퍼 앱에서도 expo-router가 중요한가?**
A. 비중이 작다. WebView 래퍼는 화면 대부분을 내부 웹앱 라우터(Next)가 담당. expo-router는 네이티브 화면을 여러 개 가진 풀 RN 앱일 때 진가가 난다. WebView 래퍼에서 네이티브 화면 비중을 늘리는 전환점에서 중요해진다.

## 주의사항 / 자주 하는 실수

- **중첩 시 이중 헤더** — 바깥 네비게이터에서 `headerShown: false` 안 하면 헤더 2개.
- **로그인 후 `push`** — 뒤로가기로 로그인 화면 복귀. `replace` 필수.
- **param을 number로 가정** — 전부 string. `Number()` 변환.
- **`useGlobalSearchParams` 남용** — 비활성 화면까지 리렌더. 기본은 `useLocalSearchParams`.
- **인증 가드에서 `isLoading` 누락** — 세션 복원 중 로그인 유저도 로그인 화면 깜빡임.
- **세션을 AsyncStorage 평문 저장** — `expo-secure-store` 사용.
- **세밀한 옵션을 expo-router 문서에서만 찾기** — 헤더/제스처/애니메이션은 아래 React Navigation 문서로.
- **웹 타깃인데 네이티브 스택 동작 가정** — 웹은 교체라 "뒤 화면 생존"이 다름. 테스트 필요.

## 참고

- [RN+Expo WebView 래퍼 아키텍처](RN-Expo-WebView래퍼-아키텍처.md) ← OTA/Hermes/Bridge/푸시
- [RN vs Flutter 비교](RN-vs-Flutter-비교.md) ← 프레임워크 선택
- [Next.js 기본](../프론트엔드/Next.js-기본.md) ← 파일 기반 라우팅 원조
- expo-router 공식: https://docs.expo.dev/router/introduction/
- Protected routes: https://docs.expo.dev/router/advanced/protected/
- React Navigation: https://reactnavigation.org
