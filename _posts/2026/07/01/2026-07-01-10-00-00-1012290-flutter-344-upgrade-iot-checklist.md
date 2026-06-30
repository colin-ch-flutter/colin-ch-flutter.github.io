---
layout: post
title: "Flutter 3.44 업그레이드 체크리스트 - IoT 앱에서 먼저 볼 것들"
description: "Flutter 3.44 업그레이드 전에 IoT 앱에서 확인해야 할 Android 플랫폼 뷰, Swift Package Manager, Impeller, Widget Previews 항목을 실제 점검 순서로 정리했다."
date: 2026-07-01
tags: [Flutter, Android, iOS, IoT, CI/CD]
comments: true
share: true
---

![Flutter 3.44 업그레이드 전 코드 점검](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

Flutter 3.44 업그레이드는 `flutter upgrade`만 치고 끝내면 안 된다. IoT 앱은 BLE, 카메라, 웹뷰, 네이티브 플러그인 의존성이 많아서 Android 플랫폼 뷰와 iOS 빌드 체인을 먼저 확인해야 한다.

## Flutter 3.44에서 IoT 앱이 먼저 봐야 할 변화

2026년 5월 20일 공개된 Flutter 3.44 기준으로 내가 먼저 본 건 네 가지다.

- Android의 Hybrid Composition++ 기본 적용
- iOS/macOS 신규 앱의 Swift Package Manager 기본 사용
- Impeller Vulkan 렌더러 개선
- Widget Previews 지원 강화

스마트홈 앱 입장에서는 전부 남의 일이 아니다. BLE 프로비저닝 화면에는 `mobile_scanner` 같은 카메라 플러그인이 붙고, 기기 상세에는 웹뷰나 지도, 차트가 들어가는 경우가 많다. iOS 쪽은 `flutter_blue_plus`, `firebase_*`, `permission_handler` 같은 플러그인 조합이 빌드 체인에 바로 영향을 준다.

처음엔 "이번 릴리스는 성능 개선 위주겠지"라고 생각했는데, 릴리스 노트를 보니 업그레이드 체크리스트를 따로 만들어야겠다는 쪽으로 마음이 바뀌었다. 특히 운영 중인 앱이면 빌드 성공보다 화면 전환, 네이티브 뷰 스크롤, BLE 권한 팝업까지 보는 게 맞다.

## Android Hybrid Composition++는 어디서 터질 수 있나

Hybrid Composition++는 Android 플랫폼 뷰 렌더링 비용을 줄이는 방향의 변경이다. 말은 좋은데, 기존 앱에서 확인할 지점은 명확하다.

- QR 코드 스캔 화면
- 웹뷰로 띄우는 이용약관, 도움말, 기기 매뉴얼
- 지도 또는 네이티브 SDK 뷰
- BLE 스캔 중 표시되는 바텀시트와 카메라 프리뷰 조합

예전에 QR 스캔 화면을 만들 때 `mobile_scanner` 프리뷰 위에 반투명 오버레이를 얹었는데, Android 12 실기기에서 터치가 한 박자 늦게 먹는 일이 있었다. 그때는 플러그인 문제라고만 생각했는데, 플랫폼 뷰가 섞이면 Flutter 위젯만 있는 화면과 다르게 봐야 한다.

업그레이드 후에는 이런 흐름을 손으로 한 번씩 밟아본다.

```bash
flutter clean
flutter pub get
flutter run --release --dart-define=CONFIG=stage
```

`stage` 환경으로 실행하는 이유는 운영 API를 건드리지 않으면서 실제 설정과 가장 비슷하게 보기 위해서다. 환경 분리는 이전에 쓴 [멀티 환경 빌드 설정]({% post_url 2026-06-16-09-36-24-592560-multi-environment-build-dev-stage-prod %}) 방식처럼 `--dart-define`으로 유지하는 게 편했다.

## iOS Swift Package Manager 전환은 바로 적용하지 않아도 된다

Flutter 3.44에서 iOS/macOS 신규 프로젝트는 Swift Package Manager를 기본 패키지 매니저로 쓴다. 기존 프로젝트가 갑자기 CocoaPods에서 SwiftPM으로 바뀌는 건 아니지만, 새 모듈을 만들거나 앱을 갈아엎는 타이밍이면 이 변화가 들어온다.

내 기준에서 바로 전환하지 않는 이유는 단순하다. IoT 앱은 네이티브 플러그인 비중이 높다.

- BLE: `flutter_blue_plus`
- 권한: `permission_handler`
- 보안 저장소: `flutter_secure_storage`
- Firebase: `firebase_core`, `firebase_crashlytics`, `firebase_remote_config`
- 로컬 알림: `flutter_local_notifications`

이 조합은 하나만 삐끗해도 `pod install` 문제가 아니라 Xcode 빌드 문제로 번진다. SwiftPM이 나쁘다는 뜻은 아니다. 다만 운영 앱에서는 "공식 기본값이 바뀌었다"와 "우리 앱에서 지금 바꿔도 된다"를 분리해야 한다.

iOS 쪽은 일단 이 정도만 체크한다.

```bash
flutter build ios --release --no-codesign
flutter test
```

빌드만 통과하면 부족하다. 실기기에서 BLE 권한 팝업, 백그라운드 복귀, Wi-Fi 프로비저닝 실패 후 재시도까지 본다. iOS 시뮬레이터에서는 BLE가 안 되기 때문에 여기서 시간을 쓰면 안 된다. 이건 몇 번 당해도 또 까먹는다.

![Flutter iOS 빌드와 네이티브 플러그인 점검](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

## Impeller Vulkan은 성능보다 화면 깨짐을 먼저 본다

Impeller는 이제 Flutter 렌더링 이야기에서 계속 따라다닌다. 3.44에서는 Vulkan 쪽 개선도 들어갔다. 나는 이런 변경이 나오면 FPS 숫자보다 화면 깨짐을 먼저 본다.

스마트홈 앱에서 문제가 잘 보이는 화면은 따로 있다.

- 실시간 전력 사용량 차트
- 온도 변화 라인 차트
- 기기 카드 리스트
- 다크모드에서 반투명 상태 뱃지가 겹치는 화면

`fl_chart`를 많이 쓰는 앱이면 차트 애니메이션이 튀는지 확인해야 한다. 특히 MQTT 메시지가 1초마다 들어오고 차트가 계속 다시 그려지는 화면은 릴리스 빌드로 봐야 한다. 디버그 빌드에서 멀쩡하다고 운영에서도 멀쩡한 건 아니었다.

나는 업그레이드 브랜치에서 임시로 이런 로그를 켜둔다.

```dart
class RenderProbe {
  static void mark(String screenName) {
    debugPrint('[render-probe] $screenName ${DateTime.now().toIso8601String()}');
  }
}
```

화면 진입 직후와 MQTT 스트림 수신 시점에 로그를 찍어두면, 프레임 드랍이 체감되는 순간과 데이터 갱신 타이밍을 맞춰보기 쉽다. 성능 측정 도구만 켜면 원인은 보이는데 재현 흐름이 사라지는 경우가 있어서, 이런 허술한 로그가 의외로 도움이 됐다.

## Widget Previews는 IoT 카드 UI에 먼저 붙여볼 만하다

Widget Previews는 운영 코드 전체를 띄우지 않고 위젯 상태를 빠르게 보는 쪽에 가깝다. IoT 앱에서는 기기 카드가 제일 좋은 후보였다.

기기 카드는 상태 조합이 많다.

- 온라인 / 오프라인
- 가열 중 / 대기 중
- 에러 발생
- 권한 없음
- 펌웨어 업데이트 필요

이걸 매번 앱 실행해서 MQTT 상태까지 맞추는 건 귀찮다. 위젯 테스트나 골든 테스트로도 커버할 수 있지만, 개발 중 눈으로 빨리 확인하는 용도는 다르다.

아래처럼 상태 모델을 작게 잘라두면 Preview든 Widget Test든 같이 쓰기 좋다.

```dart
enum BoilerStatus {
  online,
  offline,
  heating,
  error,
  firmwareRequired,
}

class BoilerCardState {
  final String name;
  final BoilerStatus status;
  final double currentTemp;
  final double targetTemp;

  const BoilerCardState({
    required this.name,
    required this.status,
    required this.currentTemp,
    required this.targetTemp,
  });
}
```

상태 모델을 UI 밖으로 빼두면 Widget Preview와 테스트가 같은 입력을 공유할 수 있다.

```dart
const previewBoilerState = BoilerCardState(
  name: '거실 보일러',
  status: BoilerStatus.heating,
  currentTemp: 22.5,
  targetTemp: 24.0,
);
```

이 구조는 테스트 시리즈에서 했던 Widget Test 흐름과도 잘 맞는다. 앱 라이프사이클과 MQTT 재연결을 검증했던 [WidgetsBinding 테스트 글]({% post_url 2026-06-29-10-00-00-938220-flutter-app-lifecycle-test-widgetsbinding-mqtt-reconnect %})처럼, 실제 통신을 빼고 상태만 밀어 넣는 식으로 화면을 고정할 수 있다.

## 업그레이드 브랜치에서 내가 쓰는 순서

Flutter 버전 업그레이드는 한 번에 많이 바꾸면 원인을 못 찾는다. 그래서 이 순서로 자른다.

```bash
git checkout -b chore/flutter-3-44-upgrade
flutter --version
flutter upgrade
flutter pub upgrade --major-versions
flutter test
flutter build apk --release --dart-define=CONFIG=stage
flutter build ios --release --no-codesign
```

여기서 `pub upgrade --major-versions`는 조심해서 쓴다. 플러그인까지 한꺼번에 올리면 Flutter 3.44 문제인지 패키지 문제인지 섞인다. 운영 앱이면 먼저 Flutter SDK만 올리고, 그 다음 패키지를 묶음별로 올리는 쪽이 덜 피곤했다.

내가 보는 최소 체크리스트는 이렇다.

```text
[ ] Android release 빌드 성공
[ ] iOS release 빌드 성공
[ ] BLE 스캔 / 연결 / 재연결 실기기 확인
[ ] QR 스캔 카메라 프리뷰 확인
[ ] 웹뷰 약관 화면 스크롤 확인
[ ] MQTT 재연결 후 UI 상태 복구 확인
[ ] fl_chart 실시간 차트 갱신 확인
[ ] Crashlytics 초기화 확인
[ ] stage 환경 dart-define 값 확인
```

근데 여기서 함정이 있다. 체크리스트가 길어질수록 팀에서는 "그냥 나중에 하자"가 나온다. 그래서 나는 플랫폼 뷰, BLE, 빌드 체인만 먼저 필수로 잡는다. 나머지는 화면별 QA에 붙여도 된다.

## 이 업그레이드는 지금 바로 해야 하나

새 프로젝트라면 Flutter 3.44로 시작하는 게 낫다. 기존 IoT 앱은 다르다. Android 플랫폼 뷰를 많이 쓰거나 iOS 플러그인이 복잡한 앱이면, 운영 배포 직전에 SDK를 올리는 건 별로다.

내 기준은 이렇다.

- 새 기능 브랜치 시작 전이면 올린다.
- 앱 심사 직전이면 미룬다.
- BLE나 QR 스캔 쪽 버그를 같이 고칠 계획이면 업그레이드 브랜치를 따로 판다.
- 테스트 커버리지가 낮으면 최소한 BLE/MQTT 핵심 플로우 수동 점검표를 만든다.

핵심은 Flutter 3.44 자체가 위험하다는 말이 아니다. IoT 앱은 네이티브와 맞닿은 면이 넓어서 업그레이드의 충격이 일반 CRUD 앱보다 크게 보일 수 있다는 뜻이다. SDK를 올릴 때는 "빌드됨"보다 "현장에서 기기 연결이 끊기지 않음"이 더 중요했다.

---

다음 편은 Flutter 3.44 업그레이드 브랜치에서 실제로 `flutter pub outdated` 결과를 보고 패키지 업그레이드 우선순위를 나누는 내용이다.

**참고 링크**
- [Flutter 3.44.0 release notes](https://docs.flutter.dev/release/release-notes/release-notes-3.44.0)
- [What's new in Flutter](https://docs.flutter.dev/release/whats-new)
- [Flutter Swift Package Manager 문서](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers)
