---
layout: post
title: "WakelockPlus로 IoT 모니터링 화면 켜진 채 유지하기"
description: " "
date: 2026-06-14
tags: [Flutter, WakelockPlus, 화면유지, IoT, 모니터링]
comments: true
share: true
---
![스마트홈 모니터링](https://images.unsplash.com/photo-1585771724684-38269d6639fd?w=800&q=80)

## 왜 화면이 꺼지면 안 되는가

IoT 대시보드나 IoT 모니터링 대시보드 제어 화면을 모니터로 쓰는 경우가 있다. 벽에 태블릿을 달아두고 실시간 기기 상태를 보여주는 용도다. 이럴 때 화면 자동 꺼짐이 동작하면 계속 켜야 하는 불편함이 생긴다.

또는 BLE 프로비저닝 과정처럼 진행 중인 작업이 있을 때 화면이 꺼지면 사용자가 다시 켜야 하는 번거로움이 있다.

## WakelockPlus 설정

```yaml
dependencies:
  wakelock_plus: ^1.3.2
```

### 앱 전체에 Wakelock 적용

`main.dart`에서 앱 시작과 함께 wakelock을 활성화한다:

```dart
// main.dart
class Sphere extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    WakelockPlus.enable(); // 앱 전체에서 화면 꺼짐 방지
    
    return GetMaterialApp(
      // ...
    );
  }
}
```

이렇게 하면 앱이 실행되는 동안 항상 화면이 켜진 상태를 유지한다.

### 특정 화면에서만 적용

항상 켜두는 게 배터리에 부담이 된다면, 특정 화면에서만 적용할 수 있다:

```dart
class IoT DeviceScadaControlController extends GetxController {
  @override
  void onInit() {
    super.onInit();
    // IoT 모니터링 대시보드 제어 화면 진입 시 wakelock 켜기
    WakelockPlus.enable();
  }

  @override
  void onClose() {
    // 화면 나갈 때 wakelock 끄기
    WakelockPlus.disable();
    super.onClose();
  }
}
```

## Wakelock 상태 확인

현재 wakelock 상태를 확인하고 토글하는 방법:

```dart
// 현재 상태 확인
final isEnabled = await WakelockPlus.enabled;
print('Wakelock 활성 여부: $isEnabled');

// 토글
if (isEnabled) {
  await WakelockPlus.disable();
} else {
  await WakelockPlus.enable();
}
```

## 앱 생명주기와 함께 관리

앱이 백그라운드로 가면 wakelock을 해제하는 게 배터리 효율에 좋다:

```dart
class WakelockManager extends GetxController with WidgetsBindingObserver {
  @override
  void onInit() {
    super.onInit();
    WidgetsBinding.instance.addObserver(this);
    WakelockPlus.enable();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:
        WakelockPlus.enable();
        break;
      case AppLifecycleState.paused:
      case AppLifecycleState.detached:
        WakelockPlus.disable();
        break;
      default:
        break;
    }
  }

  @override
  void onClose() {
    WidgetsBinding.instance.removeObserver(this);
    WakelockPlus.disable();
    super.onClose();
  }
}
```

## 배터리 소모 고려사항

화면을 항상 켜두면 배터리 소모가 크게 늘어난다. 스마트홈 앱에서는 앱 전체에 wakelock을 켰는데, 이유가 있다. IoT 기기 제어 앱 특성상 사용자가 화면을 보고 있는 중에 갑자기 꺼지면 제어 동작을 확인할 수 없기 때문이다.

배터리 절약이 중요한 앱이라면:
1. 특정 화면에서만 적용
2. 비활동 시간이 길면 자동 해제
3. 사용자가 설정에서 ON/OFF 선택 가능하게

## 플랫폼별 동작

- **Android**: `FLAG_KEEP_SCREEN_ON` 플래그 사용
- **iOS**: `UIApplication.idleTimerDisabled = true` 사용

두 플랫폼 모두 앱이 완전히 종료되면 자동으로 해제된다. 앱이 살아있는 동안만 유효하다.

---

다음 편은 테스트/데모 모드 구현이다.
