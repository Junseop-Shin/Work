# RN + Expo WebView 래퍼 아키텍처

> 2026-04-29 | RN, Expo, WebView, OTA, Hermes, 푸시

## 한 줄 요약

WebView로 기존 웹앱을 띄우고, 디바이스 기능·푸시·인증 같은 네이티브 영역만 RN 셸이 담당하는 하이브리드 구조 — 그 셸이 반드시 풀어야 할 4개 핵심 주제(브릿지 통신, OTA, Hermes, 푸시 권한)를 정리.

## 핵심 개념 — 네 개의 축

### 1. WebView ↔ Native Bridge

WebView 안 웹앱과 RN 셸은 **별도의 JS 런타임**에서 돈다(웹앱은 OS WebView 엔진, RN 셸은 Hermes). 메모리·컨텍스트가 분리되어 있어 통신은 메시지 패싱(`postMessage`)뿐.

**Web → Native:**
```js
window.ReactNativeWebView.postMessage(JSON.stringify({
  id, type: 'BLE_SCAN', payload: {...}
}));
```

**Native → Web:**
```tsx
webViewRef.current.injectJavaScript(`
  window.dispatchEvent(new MessageEvent('nativeMessage', { detail: ${JSON.stringify(msg)} }));
  true; // iOS WKWebView 메모리 가드 — 마지막 표현식 값 반환 회피
`);
```

`postMessage`는 fire-and-forget이라 양방향 RPC가 필요하면 **요청 ID + Promise 매칭**을 직접 짜야 한다.

### 2. OTA (EAS Update)

앱 = **네이티브 바이너리**(Obj-C/Java + 추가한 RN 모듈) + **JS 번들**(컴포넌트, 비즈니스 로직).

- 네이티브 바이너리 변경 → 스토어 심사 1~7일
- JS 번들만 변경 → OTA 즉시 배포 (분 단위)

**Runtime Version**으로 호환성 보장 — JS 번들과 네이티브 바이너리의 runtime version이 일치해야 OTA 적용. 새 네이티브 모듈 추가 / Expo SDK 메이저 업그레이드 시 runtime version 변경 필요.

WebView 래퍼에서는 셸이 얇아서 OTA 가치가 풀 RN 앱보다 작지만, **bridge 프로토콜 갱신·핫픽스용**으로는 여전히 유용.

### 3. Hermes 엔진

RN의 디폴트 JS 엔진(0.70+, Expo SDK 49+). JSC 대체.

- **AOT 바이트코드 컴파일** — 빌드 시점에 JS → .hbc, 런타임 파싱 없음 → 콜드 스타트 빠름
- **메모리 효율** — 모바일 환경 최적화
- **번들 크기 감소** — 바이트코드가 텍스트 JS보다 조밀

WebView 래퍼에선 **RN 셸의 JS만** Hermes가 처리. WebView 안 웹앱은 OS의 WebView 엔진(iOS WKWebView의 JSC, Android의 V8)이 처리. 즉 Hermes 영향은 셸 영역에 한정. 그래도 콜드 스타트 개선엔 도움.

Expo 디폴트라 따로 설정 불필요. 단, 라이브러리 선택 시 **Hermes 호환성** 체크는 필요(특히 오래된 RN 라이브러리, 동적 코드 실행 의존하는 것).

### 4. 푸시 알림 권한

**iOS의 까다로움 — 거절하면 끝:**
- 명시적 권한 요청 필수 (`requestPermissionsAsync`)
- **한 번 거절하면 코드로 다시 못 물어봄** → 설정 앱으로 안내
- 권한 유형: Authorized(일반), Provisional(조용한 권한, iOS 12+), Denied
- Provisional 활용: 처음엔 알림센터에 조용히 전달, 사용자가 "유지" 선택 시 정식 권한으로 승격

**Android (13+ 강화):**
- Android 12 이하 — 알림 기본 허용
- Android 13+ — `POST_NOTIFICATIONS` 권한 필수 (iOS와 비슷)
- **Notification Channels** (Android 8+) — 카테고리별로 사용자가 끄기 가능. 앱 설치 시 채널 사전 등록 필수.

## 언제 쓰나

- 기존 웹앱(React/Next)이 본체이고 양 OS 네이티브 셸을 통합하려는 경우
- 셸이 담당하는 일이 디바이스 기능·푸시·인증 정도로 한정
- 점진적 네이티브 화면 전환 가능성을 남겨두고 싶음 (Capacitor보다 RN이 유리한 이유)

**부적합:**
- 앱 화면 대부분이 네이티브 UI여야 함 (RN을 풀로 쓰는 게 맞음)
- 거의 모든 화면이 정적 콘텐츠 (PWA로 충분)
- 그래픽/애니메이션이 핵심 — RN/Flutter 풀스택 권장

## 사용법 / 예제

### 양방향 RPC 래퍼 (웹앱 측)

```ts
type BridgeMessage = { id: string; type: string; payload?: unknown; error?: string };
const pending = new Map<string, (m: BridgeMessage) => void>();

export function callNative<T>(type: string, payload?: unknown): Promise<T> {
  const id = crypto.randomUUID();
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      pending.delete(id);
      reject(new Error(`${type} timeout`));
    }, 30000);
    pending.set(id, (m) => {
      clearTimeout(timer);
      m.error ? reject(new Error(m.error)) : resolve(m.payload as T);
    });
    window.ReactNativeWebView.postMessage(JSON.stringify({ id, type, payload }));
  });
}

window.addEventListener('nativeMessage', (e: any) => {
  const m = e.detail as BridgeMessage;
  pending.get(m.id)?.(m);
  pending.delete(m.id);
});
```

### RN 셸의 메시지 라우팅

```tsx
<WebView
  ref={webViewRef}
  source={{ uri: 'https://app.example.com' }}
  originWhitelist={['https://app.example.com']}
  onMessage={(e) => {
    const msg = JSON.parse(e.nativeEvent.data);
    handleMessage(msg);
  }}
/>

async function handleMessage(msg: BridgeMessage) {
  try {
    if (msg.type === 'BLE_SCAN') {
      const devices = await scanBLE(msg.payload);
      emit({ id: msg.id, type: 'response', payload: devices });
    } else if (msg.type === 'PUSH_TOKEN_REQUEST') {
      const token = await getPushToken();
      emit({ id: msg.id, type: 'response', payload: token });
    }
  } catch (err: any) {
    emit({ id: msg.id, type: 'response', error: err.message });
  }
}

function emit(msg: BridgeMessage) {
  webViewRef.current?.injectJavaScript(`
    window.dispatchEvent(new MessageEvent('nativeMessage', { detail: ${JSON.stringify(msg)} }));
    true;
  `);
}
```

### OTA 배포

```bash
# 초기 셋업
npx expo install expo-updates
eas init

# 채널별 배포
eas update --branch production --message "Fix push token registration"

# 롤백
eas update:republish --branch production --update-id <previous-id>
```

```json
// app.json
{
  "expo": {
    "updates": { "url": "https://u.expo.dev/<project-id>" },
    "runtimeVersion": { "policy": "appVersion" }
  }
}
```

### 푸시 권한 + 토큰 → 웹앱 전달

```tsx
import * as Notifications from 'expo-notifications';
import { Platform } from 'react-native';

export async function setupPush(): Promise<string | null> {
  // Android 8+ 채널 등록
  if (Platform.OS === 'android') {
    await Notifications.setNotificationChannelAsync('transactional', {
      name: '거래 알림',
      importance: Notifications.AndroidImportance.HIGH,
    });
    await Notifications.setNotificationChannelAsync('marketing', {
      name: '마케팅',
      importance: Notifications.AndroidImportance.DEFAULT,
    });
  }

  // 권한 요청 — pre-prompt UI 띄운 후 사용자 동의 시점에 호출 권장
  const { status: existing } = await Notifications.getPermissionsAsync();
  let status = existing;
  if (status !== 'granted') {
    const r = await Notifications.requestPermissionsAsync();
    status = r.status;
  }
  if (status !== 'granted') return null;

  const token = (await Notifications.getDevicePushTokenAsync()).data;
  return token;
}

// 푸시 탭 → WebView 라우팅
Notifications.addNotificationResponseReceivedListener((res) => {
  const url = res.notification.request.content.data?.url as string | undefined;
  if (url) {
    webViewRef.current?.injectJavaScript(`window.location.href = '${url}'; true;`);
  }
});
```

## 비슷한 것과의 차이

### Bridge 통신 방식

| | RN WebView Bridge | RN Native Module (TurboModule) |
|--|--|--|
| 누가 호출 | WebView 안 웹앱 JS | RN 셸 JS |
| 직렬화 | JSON 텍스트 | JSI 직접 메모리 접근 |
| 타입 안정성 | 런타임 검증 필요 (zod 등) | TS + codegen 정적 보장 |
| 적합 | WebView 래퍼 | 풀 RN 앱 |

### 배포 채널

| | EAS Update | App/Play Store |
|--|--|--|
| 속도 | 분 단위 | 1~7일 (심사) |
| 네이티브 변경 | ❌ | ✅ |
| Apple 정책 | 4.7 가이드라인 (콘텐츠 OK, 핵심 기능 변경 금지) | 정상 심사 |

## 핵심 질의응답

**Q. 왜 RN+WebView가 Capacitor보다 나은가?**
A. (1) 생태계 크기 자체가 다름, (2) BLE/NFC 라이브러리(`react-native-ble-plx`, `react-native-nfc-manager`)가 더 검증됨, (3) 미래에 특정 화면을 네이티브 UI로 옮기고 싶을 때 RN은 그 화면을 RN 컴포넌트로 추가 가능, Capacitor는 사실상 WebView 안에 갇힘.

**Q. `postMessage` 한 번에 응답이 안 오는 이유?**
A. 단방향 fire-and-forget이라. 양방향 RPC가 필요하면 ID 매칭 + Promise 래퍼 + timeout 가드를 직접 구현해야 한다. 라이브러리에 의존하지 말고 작게 직접 짜는 게 디버깅 편함.

**Q. WebView 래퍼 앱에서 OTA가 정말 필요한가?**
A. 절반 맞다. 웹 콘텐츠는 이미 OTA(서버 호스팅). 그러나 RN 셸 로직(브릿지, 디바이스 모듈 호출, 권한 처리)은 OTA 대상. 셸이 얇을수록 가치 작지만 핫픽스용으로 보유 권장.

**Q. 자동로그인은 파일시스템에 저장하나?**
A. 아니다. 토큰은 OS의 보안 저장소(iOS Keychain, Android Keystore + EncryptedSharedPreferences)에 저장. 라이브러리: `expo-secure-store` 또는 `react-native-keychain`. 평문 저장은 보안 사고.

**Q. iOS 푸시 권한 거절된 사용자는 끝인가?**
A. 코드로 재요청 불가. 두 가지 우회: (1) `Linking.openSettings()`로 설정 앱 안내, (2) 처음부터 Provisional 권한으로 받아 알림센터에 조용히 전달 → 사용자가 "유지" 선택 시 정식 권한 승격. Pre-prompt(자체 안내 화면) 후 OS 다이얼로그 띄우는 게 거절률 줄임.

**Q. Hermes는 따로 설정할 게 있나?**
A. 거의 없음. Expo 디폴트라 활성화된 채로 두면 됨. 단 라이브러리 선택 시 Hermes 호환성 체크 — 동적 `eval`/`Function` 의존하는 오래된 라이브러리는 호환 안 될 수 있음. 최근 라이브러리는 거의 다 호환.

**Q. 푸시 페이로드로 WebView를 특정 URL로 보내려면?**
A. 푸시 데이터에 `url` 필드 담아서 전송 → `addNotificationResponseReceivedListener`로 탭 이벤트 잡고 → `injectJavaScript`로 `window.location.href` 변경. 또는 React Router/Next의 deep link API 사용.

## 주의사항 / 자주 하는 실수

**Bridge:**
- 요청 ID 없이 type만 매칭 → 동시 요청 시 응답 섞임
- `injectJavaScript`에 `true;` 누락 → iOS 메모리 경고 / 잠재 크래시
- `originWhitelist` 미설정 → 외부 악성 URL이 네이티브 인터페이스 노출
- 큰 바이너리(이미지 등)를 base64로 보냄 → 메모리 폭발. `file://` URL이나 디스크 경유로 우회
- error 케이스 시 pending Map cleanup 누락 → Promise 영원히 pending + 메모리 누수
- 고빈도 이벤트(BLE 스캔 결과) throttle 없이 emit → 메시지 큐 폭주

**OTA:**
- `runtimeVersion` 정책 미설정 → 의도치 않은 OTA 차단
- 네이티브 변경 후 OTA만 시도 → 적용 안 됨
- 채널 분리 안 하고 production에 직접 테스트 → staging 채널 통해 검증 후 배포가 정석
- Apple 4.7 가이드라인 위반 (OTA로 핵심 기능 토글) → 앱 거절 위험

**Hermes:**
- 오래된 라이브러리의 동적 코드 실행 비호환 → 채택 전 GitHub issue 확인
- Chrome DevTools 대신 React DevTools / Flipper 사용 (디버깅 도구 차이)

**푸시:**
- 앱 첫 실행에 즉시 OS 다이얼로그 → 거절률 50%+. Pre-prompt 후 컨텍스트 안에서 요청
- iOS 거절된 사용자에 안내 UX 누락 → 영구히 못 받음
- Android 채널 미등록 → 사용자가 마케팅만 끄려 해도 푸시 전체 차단해야 함
- 토큰 갱신 처리 누락 → 토큰은 영구 아님. `addPushTokenListener`로 갱신 추적 + 백엔드 재등록
- 포그라운드 알림 표시 처리 누락 → iOS 포그라운드 시 기본적으로 안 뜸. `setNotificationHandler`로 명시적 처리

## 참고

- react-native-webview: https://github.com/react-native-webview/react-native-webview
- EAS Update: https://docs.expo.dev/eas-update/introduction/
- Apple Review Guideline 4.7: https://developer.apple.com/app-store/review/guidelines/#hardware-compatibility
- Hermes 엔진: https://hermesengine.dev
- expo-notifications: https://docs.expo.dev/versions/latest/sdk/notifications/
- Notification Channels (Android): https://developer.android.com/develop/ui/views/notifications/channels
