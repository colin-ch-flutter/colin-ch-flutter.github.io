---
layout: post
title: "Flutter integration_test MQTT 상태 경합 검증 - 늦게 온 IoT 이벤트 버리기"
description: "Flutter integration_test에서 MQTT 늦은 이벤트가 Flutter 스마트홈 화면의 최신 보일러 상태를 덮어쓰는 문제를 재현하고, sequence와 commandId로 오래된 IoT 상태를 걸러내는 방법을 정리했다."
date: 2026-08-23
tags: [Flutter, Dart, MQTT, mqtt5_client, integration_test, IoT, 스마트홈]
comments: true
share: true
---
![Flutter integration_test MQTT 늦은 이벤트와 IoT 상태 경합 검증](https://images.unsplash.com/photo-1518770660439-4636190af475?w=1200&q=80)

Flutter integration_test에서 MQTT 연결 성공과 ACK 수신만 확인하면 부족하다. 스마트홈 앱에서는 사용자가 보일러를 끈 직후, 네트워크에 남아 있던 이전 `on` 상태가 늦게 도착해 화면이 다시 켜지는 문제가 있었다. 결론은 **상태 이벤트에 순서 정보를 넣고, 테스트에서 지연된 이벤트를 의도적으로 주입해야 한다**는 것이다.

## 화면이 잠깐 되돌아간 이유

처음 구현은 토픽을 구독하면 도착한 payload를 그대로 `currentState`에 넣었다. MQTT는 전송 순서를 앱 화면까지 보장하지 않는다. 특히 Wi-Fi가 불안정한 IoT 환경에서는 아래처럼 이벤트가 뒤집힐 수 있다.

| 이벤트 | 실제 발생 순서 | 앱 도착 순서 | 잘못된 결과 |
|---|---:|---:|---|
| 보일러 ON | 1 | 2 | 이전 상태가 늦게 반영됨 |
| 보일러 OFF | 2 | 1 | 최신 상태가 사라짐 |

단순히 `pumpAndSettle()`을 여러 번 호출해서는 이 버그가 재현되지 않았다. Fake MQTT에서 첫 이벤트만 늦추는 장치가 필요했다.

## sequence를 상태에 함께 보낸다

기기나 중계 서버가 `sequence`를 증가시켜 보내도록 하고, Repository는 현재보다 작은 이벤트를 버린다. 실제 코드에서는 서버가 발급하는 `commandId`를 함께 쓰면 재시작 뒤에도 더 안전하다.

```dart
class DeviceTelemetry {
  const DeviceTelemetry({required this.sequence, required this.isOn});

  final int sequence;
  final bool isOn;
}

class DeviceStateStore {
  int _lastSequence = -1;
  bool isOn = false;

  void apply(DeviceTelemetry event) {
    if (event.sequence <= _lastSequence) return;
    _lastSequence = event.sequence;
    isOn = event.isOn;
  }
}
```

## integration_test에서 늦은 이벤트를 만든다

테스트 앱에는 운영 코드에 영향을 주지 않는 `TestMqttProbe`를 주입했다. `emitLater()`로 오래된 ON 이벤트를 늦게 보내고, OFF 상태가 유지되는지 확인한다.

```dart
testWidgets('오래된 MQTT 이벤트가 최신 상태를 덮어쓰지 않는다', (tester) async {
  final probe = TestMqttProbe();
  await launchTestApp(mqtt: probe);

  await tester.tap(find.byKey(const Key('boiler-off')));
  probe.emit(DeviceTelemetry(sequence: 2, isOn: false));
  probe.emitLater(
    const DeviceTelemetry(sequence: 1, isOn: true),
    const Duration(milliseconds: 300),
  );

  await tester.pump(const Duration(milliseconds: 350));
  expect(find.byKey(const Key('boiler-off')), findsOneWidget);
  expect(find.text('꺼짐'), findsOneWidget);
});
```

처음에는 테스트가 계속 통과해서 구현이 맞다고 생각했다. 알고 보니 `emitLater()`의 Future를 기다린 뒤 상태를 검사하고 있었다. 그러면 늦은 이벤트가 이미 처리된 뒤라서 의미가 없다. 실제 사용자 타이밍처럼 최신 이벤트를 먼저 반영하고, 그 사이 화면을 한 번 확인한 뒤 오래된 이벤트를 보내야 했다.

## 테스트에서 꼭 확인할 기준

- 같은 `sequence`는 두 번 반영하지 않는다.
- 더 작은 `sequence`는 로그만 남기고 상태를 바꾸지 않는다.
- 앱 재연결 뒤 서버의 최신 snapshot은 현재 sequence를 갱신한다.
- 테스트 종료 시 MQTT Stream과 Timer를 닫아 다음 테스트에 이벤트가 새지 않는다.

짧게 정리하면 Flutter integration_test의 MQTT 검증은 “메시지를 받았는가”에서 끝나면 안 된다. 늦게 온 IoT 상태가 최신 화면을 덮어쓰는 경합까지 Fake로 재현해야 한다. `sequence` 비교는 코드가 작지만, 보일러가 꺼졌는데 화면만 켜져 있는 식의 신뢰도 낮은 UI를 막아준다.
