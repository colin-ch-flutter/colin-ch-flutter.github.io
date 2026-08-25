---
layout: post
title: "Flutter integration_test 예측 뒤로가기 - PopScope에서 IoT 구독 정리 검증"
description: "Flutter integration_test에서 Android 예측 뒤로가기와 PopScope의 취소·완료 콜백을 검증하고, IoT 상세 화면의 MQTT 구독이 남지 않는지 테스트하는 방법을 정리했다."
date: 2026-08-26
tags: [Flutter, Dart, integration_test, MQTT, IoT, Android]
comments: true
share: true
---

![Flutter integration_test Android 예측 뒤로가기와 IoT 화면 전환](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Flutter integration_test에서 뒤로가기 버튼을 눌러 화면이 닫히는지만 보면 부족하다. Android 예측 뒤로가기는 취소될 수도 있어 `PopScope`의 `didPop`과 MQTT 구독 정리를 함께 검증해야 한다.

## `WillPopScope`에서 `PopScope`로 바뀐 지점

처음엔 `WillPopScope`에서 `onWillPop`이 `true`면 끝이라고 생각했다. 화면을 나가는 리소스는 `onPopInvokedWithResult`의 `didPop`을 기준으로 정리한다.

```dart
class DeviceDetailPage extends StatelessWidget {
  const DeviceDetailPage({super.key, required this.controller});

  final DeviceController controller;

  @override
  Widget build(BuildContext context) {
    return PopScope<void>(
      canPop: !controller.hasPendingCommand,
      onPopInvokedWithResult: (didPop, result) {
        // 예측 뒤로가기가 취소되면 구독을 끊지 않는다.
        if (didPop) controller.disposeDeviceSubscription();
      },
      child: DeviceControlView(controller: controller),
    );
  }
}
```

`canPop`이 false인 동안에는 보일러 명령이 ACK를 기다리는 상태라 뒤로가기를 막는다. 예측 시작과 pop 완료를 같은 것으로 취급하지 않는 게 핵심이다.

| 상황 | `didPop` | MQTT |
|---|---:|---|
| 손가락을 끌다 취소 | false | 기존 구독 유지 |
| 명령 대기 없이 나감 | true | 상세 토픽 unsubscribe |

## integration_test에서 취소와 완료를 나눠 확인하기

테스트 앱에는 `maybePop()`을 호출하는 테스트 전용 버튼을 두고, Fake MQTT 서비스는 구독·해제 횟수를 기록한다.

```dart
testWidgets('뒤로가기 취소 시 MQTT 구독을 유지한다', (tester) async {
  final mqtt = await pumpDevice(tester);
  expect(mqtt.subscribeCount, 1);
  await tester.tap(find.byKey(const Key('cancel-pop')));
  await tester.pump();
  expect(find.byKey(const Key('device-detail')), findsOneWidget);
  expect(mqtt.unsubscribeCount, 0);
});

testWidgets('뒤로가기 완료 시 상세 MQTT 구독을 해제한다', (tester) async {
  final mqtt = await pumpDevice(tester);
  await tester.tap(find.byKey(const Key('complete-pop')));
  await tester.pumpAndSettle();
  expect(find.byKey(const Key('device-list')), findsOneWidget);
  expect(mqtt.unsubscribeCount, 1);
});
```

실패 원인은 `dispose()`에서 무조건 unsubscribe를 호출한 것이었다. 라우트 종료와 Controller의 `onClose()` 중 한 곳만 구독 정리의 소유자가 되도록 바꾸니 중복 해제가 사라졌다.

실기기에서는 예측 뒤로가기를 한 번 확인하되, OS 애니메이션 모양까지 성공 조건으로 넣지는 않는다. `didPop` 기준으로 정리하면 취소된 제스처 뒤 MQTT 이벤트가 새지 않는다.
