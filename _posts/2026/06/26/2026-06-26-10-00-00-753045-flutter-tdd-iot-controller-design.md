---
layout: post
title: "Flutter TDD 실전 — 테스트 먼저 짜면 IoT Controller 설계가 달라진다"
description: "테스트 코드를 먼저 작성하는 TDD를 Flutter IoT Controller에 적용한 실전 기록. Red-Green-Refactor 한 사이클로 BleController 의존성 설계가 어떻게 바뀌는지, TDD가 어울리는 레이어와 억지로 쓰면 오히려 느려지는 레이어를 구분해서 정리했다."
date: 2026-06-26
tags: [Flutter, Dart, GetX, CleanArchitecture, BLE]
comments: true
share: true
---

# Flutter TDD 실전 — 테스트 먼저 짜면 IoT Controller 설계가 달라진다

![Flutter TDD 테스트 주도 개발 Red-Green-Refactor](https://images.unsplash.com/photo-1517694712202-14dd9538aa97?w=800&q=80)

TDD를 처음 들었을 때 솔직히 말도 안 된다고 생각했다. 코드도 없는데 뭘 테스트한다는 건지. 그런데 IoT 앱 테스트 시리즈를 진행하면서 생각이 바뀌었다. 테스트를 먼저 쓰면 클래스 설계가 달라진다. 특히 BleController 같은 외부 의존성이 많은 코드는, 테스트 코드를 먼저 써야만 주입 구조가 제대로 잡혔다.

## TDD가 뭔지는 알겠는데, IoT 앱에서 될까?

이론은 간단하다. **Red → Green → Refactor** 세 단계를 반복한다.

1. **Red**: 실패하는 테스트를 먼저 작성
2. **Green**: 테스트를 통과시키는 최소한의 코드 작성
3. **Refactor**: 구조 개선 (테스트는 계속 Green 유지)

문제는 "IoT 앱에서 TDD가 현실적인가"다. BLE 연결, MQTT 메시지, 보일러 상태 응답 — 전부 실기기와 외부 서버가 필요한 것들이다. 근데 지금까지 이 시리즈에서 만든 Fake 인터페이스들이 여기서 빛을 발한다.

[BLE Wrapper 패턴]({% post_url 2026-06-24-10-00-00-654285-flutter-blue-plus-ble-unit-test-wrapper-isolation %})과 [MQTT Fake 서비스]({% post_url 2026-06-24-14-00-00-666630-mqtt5-client-unit-test-fake-service-isolation %})가 있으면 TDD도 충분히 가능하다.

## TDD로 BleController 새 기능 추가하기

이미 만들어진 `BleController`를 뜯어고치는 게 아니라, 처음부터 TDD 방식으로 새 기능을 추가하는 예시다. 시나리오: **"연결된 BLE 기기가 있을 때는 스캔을 시작하지 않는다."**

### Red: 실패하는 테스트 먼저

```dart
// test/controllers/ble_controller_scan_test.dart

void main() {
  late BleController controller;
  late FakeBleService fakeBle;

  setUp(() {
    fakeBle = FakeBleService();
    controller = BleController(bleService: fakeBle); // 이 생성자가 아직 없음
  });

  test('이미 연결된 기기가 있으면 스캔을 시작하지 않는다', () {
    fakeBle.simulateConnected('device-01');

    controller.startScanIfDisconnected();

    expect(fakeBle.scanStarted, isFalse);
  });

  test('연결된 기기가 없으면 스캔을 시작한다', () {
    // 기본 상태는 미연결

    controller.startScanIfDisconnected();

    expect(fakeBle.scanStarted, isTrue);
  });
}
```

실행하면 컴파일 에러부터 난다. `BleController`에 `bleService` 파라미터가 없고, `startScanIfDisconnected()`도 없다. 이 상태가 Red다.

여기서 핵심이 드러난다. 테스트를 먼저 썼기 때문에 **처음부터 `bleService`를 주입받는 구조를 강제**하게 된다. 테스트 없이 개발했다면 `BleController` 내부에서 `FlutterBluePlus`를 직접 호출하고, 나중에 테스트하려고 뜯어고쳐야 했을 것이다.

### Green: 통과하는 최소 코드

```dart
// lib/controllers/ble_controller.dart

class BleController extends GetxController {
  final IBleService _bleService;

  BleController({required IBleService bleService})
      : _bleService = bleService;

  void startScanIfDisconnected() {
    if (_bleService.connectedDevice != null) return;
    _bleService.startScan();
  }
}
```

`IBleService`와 `FakeBleService`는 시리즈 3편에서 이미 만들었다. 테스트 실행하면 Green이 된다.

### Refactor: 의미를 더 명확하게

```dart
void startScanIfDisconnected() {
  if (_bleService.isConnected) return; // connectedDevice != null보다 의미 명확
  _bleService.startScan();
}
```

`FakeBleService`에 `isConnected` getter를 추가하고 테스트 재실행 — Green 유지. 이게 Refactor 단계다. 테스트가 있으니 구조를 바꿔도 겁이 없다.

![Flutter 테스트 코드 의존성 주입 설계](https://images.unsplash.com/photo-1555949963-ff9fe0c870eb?w=800&q=80)

## 실제로 겪은 변화 — NotificationController 삽질

TDD 방식으로 `NotificationController`를 새로 만들다가 바로 막혔다.

```dart
setUp(() {
  controller = NotificationController(); // Firebase 초기화가 여기서 터짐
});
```

기존 구조는 `GetIt`에서 `FirebaseMessaging.instance`를 직접 가져왔다. 생성자에서 Firebase가 이미 초기화되어 있어야 하는데, 테스트 환경에는 Firebase가 없으니 터진다.

테스트 없이 개발했다면 실기기에서 돌리기 전까지 이 문제를 몰랐을 것이다. TDD라서 Day 1에 발견했다.

결국 `INotificationService`를 추출하고 주입 구조로 바꿨다:

```dart
class NotificationController extends GetxController {
  final INotificationService _notificationService;

  NotificationController({required INotificationService notificationService})
      : _notificationService = notificationService;
}
```

코드 수정 10분. 이후 테스트 구조가 깔끔해졌고, 나중에 `FakeNotificationService`로 FCM 수신 시나리오까지 통제할 수 있게 됐다.

## TDD가 잘 맞는 레이어 vs 억지로 쓰면 느려지는 레이어

이걸 모르면 TDD가 오히려 발목을 잡는다.

| 레이어 | TDD 적합성 | 이유 |
|--------|-----------|------|
| UseCase / Repository | 높음 | 입출력 명확, UI 없음 |
| Controller 비즈니스 로직 | 높음 | Fake로 완전 격리 가능 |
| 유틸·파서 (순수 함수) | 높음 | 외부 의존성 없음 |
| Widget Test | 중간 | UI 없는데 픽셀 검증 어색 |
| Golden Test | 낮음 | 디자인 시안 없이 골든 파일 못 만듦 |
| Integration / Patrol E2E | 낮음 | 앱이 실행돼야 테스트를 쓸 수 있음 |

Flutter 앱에서 TDD는 **단위 테스트 레이어에서만** 현실적이다. Widget / Integration 쪽은 구현 후 테스트 추가가 맞다.

## 시리즈 마무리 — 테스트 전략 한 줄 요약

11편을 쓰면서 결국 도달한 결론은 하나다. 테스트는 코드 검증이 아니라 설계 도구다.

BLE 시리즈 초반에 "연결만 되면 되지, 테스트는 나중에"라고 생각했던 코드들이 나중에 다 발목을 잡았다. 처음부터 주입 구조로 잡았더라면 훨씬 빨랐을 것이다.

[Unit Test 시리즈 1편 — Repository 레이어 테스트]({% post_url 2026-06-23-10-00-00-629595-flutter-unit-test-repository-layer-start %})부터 이번 TDD 편까지, 쌓아온 Fake 인터페이스가 전부 여기서 쓰였다. 처음엔 귀찮아 보였던 추상화가 테스트를 가능하게 만든다.

다음 시리즈는 Flutter Riverpod 상태관리 실전으로 넘어갈 예정이다.

---

**참고 링크**
- [Flutter 공식 테스팅 문서](https://docs.flutter.dev/testing)
- [Patrol 공식 사이트](https://patrol.leancode.co/)
- [ArcTouch — TDD with Flutter](https://arctouch.com/blog/flutter-test-driven-development)
