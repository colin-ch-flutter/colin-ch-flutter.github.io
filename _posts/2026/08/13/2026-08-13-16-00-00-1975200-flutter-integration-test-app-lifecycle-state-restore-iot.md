---
layout: post
title: "Flutter integration_test 앱 생명주기 테스트 - MQTT 재연결과 상태 복원 검증"
description: "Flutter integration_test에서 앱을 백그라운드로 보냈다가 복귀할 때 MQTT 재연결과 IoT 제어 상태가 복원되는지 검증하는 실전 패턴을 정리했다."
date: 2026-08-13
tags: [Flutter, Dart, integration_test, MQTT, mqtt5_client, IoT, CI/CD]
comments: true
share: true
---

![Flutter integration_test 앱 생명주기와 MQTT 상태 복원](/assets/images/flutter-integration-test-parallel-ci-iot.png)

Flutter integration_test에서 버튼을 누르는 흐름만 통과해도 앱이 실제로 안전한 건 아니다. IoT 앱은 백그라운드 전환 뒤 MQTT 연결이 끊기고, 복귀했을 때 마지막 기기 상태를 다시 가져와야 한다. 이 생명주기를 테스트하지 않으면 사용자가 앱을 다시 열었을 때 오래된 보일러 상태를 보게 된다.

처음엔 `AppLifecycleState.resumed`에서 `connect()`만 호출했다. 그런데 복귀 이벤트가 짧은 시간에 두 번 들어오면서 MQTT 클라이언트가 이미 연결 중인데 또 연결을 시작하는 문제가 생겼다.

## 복귀 로직에 중복 연결 방지 조건 넣기

테스트가 가능하려면 생명주기 감시 코드와 MQTT 서비스를 분리하고, 연결 중 상태를 외부에서 확인할 수 있어야 한다.

```dart
class MqttLifecycleService with WidgetsBindingObserver {
  MqttLifecycleService(this.client, this.loadLatestState);

  final MqttService client;
  final Future<void> Function() loadLatestState;
  bool _reconnecting = false;

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) {
      _restore();
    }
  }

  Future<void> _restore() async {
    if (_reconnecting || client.isConnected) return;
    _reconnecting = true;
    try {
      await client.connect();
      await loadLatestState();
    } finally {
      _reconnecting = false;
    }
  }
}
```

핵심은 연결 성공만 확인하지 않고 `loadLatestState()`까지 기다리는 것이다. 그래야 화면의 로딩 표시가 끝난 뒤 서버의 최신 온도와 전원 상태를 검증할 수 있다.

## integration_test에서 백그라운드 전환 흉내 내기

`integration_test`는 실제 앱 프로세스를 띄우므로 `WidgetsBindingObserver`에 이벤트를 직접 넣기보다, 테스트 전용 debug 메뉴를 통해 같은 복원 함수를 호출하게 만들었다.

```dart
testWidgets('앱 복귀 후 MQTT 재연결과 최신 상태를 반영한다', (tester) async {
  await tester.pumpWidget(const TestApp());

  await tester.tap(find.byKey(const Key('debug-background-button')));
  await tester.pump(const Duration(milliseconds: 300));
  await tester.tap(find.byKey(const Key('debug-resume-button')));

  await tester.pumpAndSettle(const Duration(seconds: 2));

  expect(find.text('MQTT 연결됨'), findsOneWidget);
  expect(find.text('보일러 22°C'), findsOneWidget);
});
```

실제 CI에서는 MQTT 브로커에 붙이지 않고 `FakeMqttService`를 주입한다. 이 테스트의 목적은 네트워크 품질이 아니라 생명주기 이벤트와 상태 복원 순서이기 때문이다.

| 검증 대상 | 실패하기 쉬운 조건 | 테스트 기준 |
|---|---|---|
| 중복 연결 | resumed 이벤트 연속 발생 | connect 호출 1회 |
| 상태 복원 | 연결 후 조회 누락 | 최신 기기 상태 표시 |
| 로딩 종료 | 예외 뒤 플래그 미복구 | 재시도 가능 상태 |

## 해보니 놓치기 쉬웠던 부분

`pumpAndSettle()`만 믿으면 MQTT 스트림이 계속 살아 있는 테스트에서 영원히 끝나지 않을 수 있다. 타임아웃을 명시하고, Fake 서비스에서 연결 완료 Future를 제어하는 편이 안전하다. 또 Android 백그라운드 전환과 iOS 복귀 타이밍은 다르므로 이 테스트 하나로 OS 동작 전체를 보장할 수는 없다.

솔직하게 정리하면, integration_test에서 중요한 건 화면 클릭 수가 아니다. 앱이 끊겼다가 돌아왔을 때 연결 중복을 막고, 최신 상태를 다시 읽고, 실패 후 재시도 가능한 상태로 남는지를 확인하는 것이다.
