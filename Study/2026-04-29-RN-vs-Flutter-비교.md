# React Native vs Flutter 비교

> 2026-04-29 | RN, Flutter, 모바일, 크로스플랫폼

## 한 줄 요약

RN은 OS의 네이티브 위젯을 JS로 조작하고, Flutter는 자체 엔진(Skia/Impeller)으로 모든 픽셀을 직접 그린다 — 이 렌더링 차이가 두 프레임워크의 모든 트레이드오프의 근원이다.

## 핵심 개념

### RN — Native Bridge / JSI

- JS(또는 TS)로 작성한 컴포넌트가 **OS의 네이티브 위젯**(iOS UIView, Android android.view.View)으로 변환되어 화면에 박힘
- `<Button>` 컴포넌트는 iOS에선 진짜 `UIButton`, Android에선 진짜 `android.widget.Button`
- 초창기는 JSON 기반 비동기 Bridge였지만, 신 아키텍처(Fabric + TurboModules + JSI)에서는 JSI(JavaScript Interface)로 동기 호출 가능 — 성능 격차 좁힘

### Flutter — 자체 렌더링 엔진

- Dart로 작성한 위젯이 Flutter 엔진(과거 Skia, 현재 Impeller로 전환 중)을 통해 **캔버스에 직접 픽셀 단위로 그림**
- OS의 네이티브 위젯을 사용하지 않음 — 버튼·텍스트·스크롤 모두 Flutter가 자기가 그림
- 결과: OS와 무관한 일관된 픽셀, 자체 디자인 시스템 강제 가능

### 결과적으로 따라오는 차이

| 측면 | RN | Flutter |
|---|---|---|
| 새 OS UI 반영 | OS 업데이트만으로 자동 | Flutter SDK 업데이트 + 앱 재배포 필요 |
| 양 OS에서 픽셀 일치 | 다르게 보임 (OS-native) | 동일 |
| 바이너리 크기 | 작음 (OS 위젯 활용) | 큼 (엔진 +5~10MB 번들) |
| 콜드 스타트 | OS 위젯 초기화 시간 | Flutter 엔진 초기화 시간 |
| 60fps 인터랙션 | 충분 (Hermes + 신 아키텍처) | 우수 (jank 적음) |

## 언제 쓰나

### React Native가 적합한 경우

- 웹 스택이 React/TS 기반 — 언어/패러다임 일치, 비즈니스 로직 공유 가능
- iOS는 iOS답게, Android는 Android답게 보여도 OK (또는 그게 더 나음)
- BLE/NFC 등 검증된 디바이스 라이브러리 생태계가 필요
- 폼/입력 위주 앱, 복잡한 그래픽 요구 적음
- WebView 래퍼 패턴(웹앱 + 디바이스 브릿지)을 고려하는 경우

### Flutter가 적합한 경우

- 양 OS에서 **픽셀 단위 일치** + **브랜드 디자인 시스템** 강제가 핵심 가치
- 복잡한 애니메이션/그래픽/커스텀 인터랙션이 많은 앱 (게임 제외)
- 데스크톱/임베디드까지 한 코드베이스로 확장 계획
- Hot Reload 같은 빠른 개발 사이클이 핵심 가치
- 팀이 JS/TS보다 정적 타입 언어(Dart)에 거부감 없음

### 둘 다 부적합 — 풀 네이티브가 나은 경우

- AR/Vision Pro / iOS-only 신기술 즉시 활용
- 게임 (Unity/Unreal이 정답)
- 백그라운드 작업이 매우 복잡하고 OS-specific (CoreLocation 백그라운드 모드, Android Foreground Service 정밀 제어 등)

## 사용법 / 예제

### RN — 같은 컴포넌트 코드, OS별 다른 결과

```tsx
import { Button, View, Text } from 'react-native';

export function Login() {
  return (
    <View>
      <Text>로그인</Text>
      <Button title="확인" onPress={() => {}} />
    </View>
  );
}
// iOS: UILabel + UIButton로 렌더
// Android: TextView + Button(Material)로 렌더
// → 두 OS에서 다르게 보임 (각 OS 디자인 시스템 따름)
```

### Flutter — 같은 위젯 코드, OS 무관 동일 결과

```dart
class Login extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('로그인'),
        ElevatedButton(
          onPressed: () {},
          child: Text('확인'),
        ),
      ],
    );
  }
}
// iOS·Android 둘 다 Material Design 버튼으로 동일하게 그려짐
// (CupertinoButton 쓰면 iOS 스타일로 그릴 수도 있지만 OS와 무관하게 Flutter가 그림)
```

## 비슷한 것과의 차이

### RN vs Flutter

| | React Native | Flutter |
|--|--|--|
| 언어 | JS / TS | Dart |
| 렌더링 | OS 네이티브 위젯 | 자체 엔진(Skia/Impeller) |
| 학습곡선 (웹 React 경험자) | 낮음 | 새 언어 + 새 패러다임 |
| 디바이스 라이브러리 (BLE, NFC) | 더 검증됨 | 따라잡는 중 |
| 디자인 일관성 | OS-native (다름) | 픽셀 일치 |
| 바이너리 크기 | 작음 | 큼 (+5~10MB) |
| Hot Reload | Fast Refresh | 더 빠르고 부드러움 |
| 웹앱 코드 공유 | RN-Web/Solito로 가능 | Flutter Web 별도, 성숙도 낮음 |
| 채택 사례 | Discord, Shopify, Coinbase | Google Pay, BMW, Alibaba |

### RN vs Capacitor (WebView 래퍼 비교)

| | RN + WebView | Capacitor |
|--|--|--|
| 본질 | RN 셸 + 내부 WebView | 웹앱 + 네이티브 셸 (PWA-first) |
| 셋업 복잡도 | 중간 (Metro, JS 번들) | 낮음 (웹 빌드 + 셸 래핑) |
| 디바이스 라이브러리 | RN 생태계 (방대) | Capacitor 플러그인 (적당) |
| 점진적 네이티브 전환 | 가능 (RN 화면 추가) | 어려움 (WebView 안에 갇힘) |
| BLE/NFC | `react-native-ble-plx`, `react-native-nfc-manager` (성숙) | `@capacitor-community/*` (덜 성숙) |
| 적합 시나리오 | 미래에 네이티브 화면 가능성 / 디바이스 기능 무거움 | 순수 WebView 래퍼 / 가벼운 디바이스 기능 |

## 핵심 질의응답

**Q. RN과 Flutter의 가장 본질적 차이는?**
A. 화면을 그리는 방식. RN은 OS 네이티브 위젯을 조작, Flutter는 자체 엔진으로 픽셀을 직접 그림. 이 한 가지에서 모든 트레이드오프(OS UI 반영 속도, 바이너리 크기, 디자인 일관성, 라이브러리 성격)가 파생된다.

**Q. 우리 앱 같은 폼/입력 + 디바이스 기능 의존(BLE/NFC/카메라/푸시)에 어느 쪽이 유리?**
A. RN. 이유는 (1) 웹 스택이 React/Next/TS라 언어 일치, (2) BLE/NFC 라이브러리가 RN 쪽이 더 검증됨, (3) 디자인 일관성 강제 욕구가 없음, (4) 복잡한 애니메이션 없음 → Flutter의 강점이 무력화됨.

**Q. 자동로그인은 파일시스템에 저장하나?**
A. 아니다. 토큰/비밀번호는 OS의 보안 저장소(iOS Keychain, Android EncryptedSharedPreferences + Keystore)에 저장해야 한다. 파일시스템에 그냥 쓰면 보안 사고. 라이브러리: RN은 `react-native-keychain` 또는 `expo-secure-store`, Flutter는 `flutter_secure_storage`.

**Q. WebView 래퍼 앱이라면 그냥 Capacitor가 더 자연스러운 선택 아닌가?**
A. 순수 WebView 래퍼만 본다면 Capacitor가 더 목적특화되어 있어 셋업이 가볍다. 그러나 RN을 택할 합리적 이유: (1) 생태계 크기, (2) BLE/NFC 같은 고급 디바이스 라이브러리가 RN이 더 검증됨, (3) 미래에 일부 화면을 네이티브 UI로 옮길 여지를 남김.

**Q. Flutter의 "픽셀 일치"가 왜 단점이 될 수도 있나?**
A. iOS 사용자는 iOS다움(스크롤 바운스, 햅틱, 컨텍스트 메뉴, 키보드 동작)을 기대한다. Flutter는 이걸 다 직접 구현해야 하고, OS 신규 디자인이 나와도 SDK 업데이트가 있어야 따라간다. 즉 "일관성"의 대가로 "OS-native 감성"을 잃는다. RN은 반대 — 자동으로 따라가지만 두 OS에서 다르게 보인다.

**Q. 통합 관리 용이성이 1순위인데, 두 프레임워크 다 만족하지 않나?**
A. 둘 다 단일 코드베이스로 양 OS 커버하니 1순위 목표는 둘 다 충족한다. 그러나 "통합 관리"는 단순히 코드만이 아니라 **인력·도구·외부 라이브러리·기존 자산 활용**까지 포함된다. 웹 React/TS 경험을 그대로 활용 + 웹과 비즈니스 로직 공유 가능 = RN이 통합 관리 측면에서 더 우위.

## 주의사항 / 자주 하는 실수

- **렌더링 아키텍처 차이를 모른 채로 비교** — "RN이 빠르다 vs Flutter가 빠르다" 식의 단순 비교는 무의미. 자기 앱이 어떤 렌더링 특성을 요구하는지부터 정의해야 한다.
- **Flutter Web을 RN-Web과 같은 수준으로 가정** — Flutter Web은 캔버스에 그리기 때문에 SEO·접근성·웹 표준 호환에 본질적 한계. RN-Web은 진짜 DOM을 만든다.
- **"Dart 학습곡선 낮다"고 가볍게 봄** — 문법 자체는 쉽지만, Flutter 위젯 트리 패러다임(everything is a widget, BuildContext, InheritedWidget, Element vs Widget vs RenderObject)은 React 멘탈모델과 다름. 적응 시간 필요.
- **기존 네이티브 앱을 한꺼번에 갈아엎으려 함** — Brownfield(점진적) 통합도 가능. 둘 다 지원하지만 RN의 brownfield 도구가 더 성숙.
- **자동로그인을 SharedPreferences/AsyncStorage에 평문 저장** — 보안 사고. 반드시 Keychain/Keystore 기반 secure storage 사용.
- **BLE/NFC 등 라이브러리 성숙도 미체크** — Flutter는 이쪽이 약점이다. 채택 전에 자기가 쓸 모든 디바이스 기능 라이브러리의 GitHub issues, last commit, 사용 사례를 확인.

## 참고

- React Native 공식: https://reactnative.dev
- Flutter 공식: https://flutter.dev
- RN 신 아키텍처(Fabric/TurboModules): https://reactnative.dev/architecture/landing-page
- Capacitor: https://capacitorjs.com
- Solito (RN + Next.js 코드 공유): https://solito.dev
- Flutter Impeller 엔진: https://docs.flutter.dev/perf/impeller
