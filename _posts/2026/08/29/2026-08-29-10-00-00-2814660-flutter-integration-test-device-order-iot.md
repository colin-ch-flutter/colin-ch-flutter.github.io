---
layout: post
title: "Flutter integration_test 멀티 디바이스 동시 제어 - MQTT 메시지 순서와 stale state 검증"
description: "Flutter integration_test에서 여러 스마트홈 기기를 동시에 제어할 때 MQTT 메시지 순서가 뒤섞이거나 stale state가 화면에 남는 문제를 Fake 서비스와 상태 검증으로 잡는 방법이다."
date: 2026-08-29
tags: [Flutter, Dart, integration_test, MQTT, IoT, 스마트홈, 테스트]
comments: true
share: true
---

![Flutter integration_test 멀티 디바이스 MQTT 메시지 순서 검증](/assets/images/flutter-integration-test-resource-cleanup-iot.png)

이 그림에서 볼 부분은 앱 화면과 IoT 이벤트 흐름을 따로 검증해야 한다는 점이다.

Flutter integration_test에서 기기 두 개를 동시에 제어할 때는 최종 화면만 검사하면 부족하다. MQTT 응답이 늦게 도착하면 먼저 실행한 명령의 오래된 상태(stale state)가 최신 상태를 덮어쓸 수 있기 때문이다. 내가 잡은 기준은 `deviceId`별 최신 시각과 `commandId`별 완료 여부를 함께 확인하는 것이었다.

## 처음에는 최종 값만 검사했다

보일러와 조명을 한 번에 켜는 테스트를 만들면서 처음에는 아래처럼 화면에 `켜짐`이 보이는지만 확인했다.

```dart
await tester.tap(find.text('전체 켜기'));
await tester.pumpAndSettle();

expect(find.text('보일러 켜짐'), findsOneWidget);
expect(find.text('조명 켜짐'), findsOneWidget);
```

이 테스트는 Fake MQTT가 응답을 등록 순서대로 보내는 동안에는 계속 통과했다. 실제 장치에서는 달랐다. 보일러 응답은 80ms, 조명 응답은 20ms 뒤에 도착하도록 만들자 이벤트 순서가 바뀌었고, Controller가 마지막으로 받은 payload를 전체 기기 상태에 적용하면서 조명 카드에 보일러 상태가 잠깐 표시됐다.

여기서 헷갈렸던 건 MQTT 메시지가 정상적으로 도착했다는 사실과 올바른 기기에 적용됐다는 사실이 서로 다르다는 점이다.

## 메시지 순서를 Fake에서 의도적으로 뒤집는다

실제 브로커를 붙이면 테스트가 느리고 재현도 어렵다. Fake 서비스에 응답 지연과 순서 뒤집기를 넣어야 한다. 아래 코드는 두 명령을 받고 응답 순서를 바꾸는 핵심 부분이다.

```dart
class FakeMqttService {
  final List<Map<String, dynamic>> published = [];
  final StreamController<Map<String, dynamic>> _events =
      StreamController<Map<String, dynamic>>.broadcast();

  Stream<Map<String, dynamic>> get events => _events.stream;

  Future<void> publishCommand({
    required String deviceId,
    required String commandId,
    required bool power,
  }) async {
    published.add({
      'deviceId': deviceId,
      'commandId': commandId,
      'power': power,
    });
  }

  void emitOutOfOrder() {
    _events.add({
      'deviceId': 'light-01',
      'commandId': 'cmd-light-2',
      'power': true,
      'updatedAt': 200,
    });
    _events.add({
      'deviceId': 'boiler-01',
      'commandId': 'cmd-boiler-1',
      'power': true,
      'updatedAt': 100,
    });
  }

  Future<void> dispose() => _events.close();
}
```

Controller에서는 전체 상태를 한 변수로 덮어쓰지 않고 기기 ID를 키로 사용한다. 또한 서버 시각이 이전 값보다 작으면 이벤트를 버린다. 장치의 시계가 완전히 신뢰되지 않는 환경이라면 `updatedAt` 대신 서버가 발급한 sequence를 쓰는 편이 낫다.

```dart
void onDeviceEvent(Map<String, dynamic> event) {
  final deviceId = event['deviceId'] as String;
  final incomingTime = event['updatedAt'] as int;
  final current = states[deviceId];

  if (current != null && incomingTime < current.updatedAt) {
    return;
  }

  states[deviceId] = DeviceState(
    deviceId: deviceId,
    power: event['power'] as bool,
    updatedAt: incomingTime,
  );
}
```

## integration_test에서는 세 가지를 같이 확인한다

테스트는 화면, 발행 payload, 최종 상태를 모두 검사해야 버그가 어디서 생겼는지 알 수 있다.

| 검사 대상 | 확인 내용 | 잡히는 문제 |
|---|---|---|
| `published` | 기기별 `commandId`가 한 번씩만 발행됐는가 | 중복 publish, 잘못된 deviceId |
| 이벤트 스트림 | 순서가 바뀌어도 오래된 이벤트를 버리는가 | stale state |
| 화면 | 보일러와 조명 카드가 각자 맞는 상태인가 | 전체 상태 덮어쓰기 |

```dart
testWidgets('out-of-order MQTT events keep device state isolated',
    (tester) async {
  final mqtt = FakeMqttService();
  await tester.pumpWidget(TestApp(mqtt: mqtt));

  await tester.tap(find.text('전체 켜기'));
  mqtt.emitOutOfOrder();
  await tester.pump();

  expect(mqtt.published.map((e) => e['deviceId']),
      containsAll(<String>['boiler-01', 'light-01']));
  expect(find.text('보일러 켜짐'), findsOneWidget);
  expect(find.text('조명 켜짐'), findsOneWidget);

  await mqtt.dispose();
});
```

`pumpAndSettle()`을 무조건 사용하지 않은 이유도 있다. MQTT 재연결 Timer나 주기적인 상태 polling이 남아 있으면 settle이 끝나지 않거나, 실제로는 계속 변하는 화면을 우연히 한 시점에만 읽게 된다. 이벤트를 넣은 뒤 `pump()`으로 한 프레임만 진행하고, 필요한 경우 `FakeAsync`로 Timer를 정확히 이동시키는 방식이 재현성이 좋았다.

## 주의할 기준

- `deviceId` 없는 공통 상태 이벤트는 개별 카드에 적용하지 않는다.
- 같은 명령의 ACK가 두 번 와도 완료 콜백은 한 번만 실행한다.
- `updatedAt`의 단위가 초와 밀리초로 섞이지 않게 Fake와 실제 adapter의 타입을 맞춘다.
- 테스트 종료 시 MQTT Stream, Timer, GetX Controller를 닫는다. 그렇지 않으면 다음 테스트가 이전 이벤트를 받는다.
- 메시지 순서 보장은 애플리케이션이 아니라 브로커·디바이스 조합에 따라 달라지므로 실제 장치 테스트를 완전히 대체하지 않는다.

솔직하게 정리하면, 멀티 디바이스 테스트의 핵심은 “두 카드가 켜졌는가”가 아니었다. 서로 다른 `deviceId`의 이벤트가 섞여도 각 상태가 독립적으로 유지되고, 오래된 응답이 최신 화면을 덮지 않는지를 검증하는 것이었다. 이 두 조건을 Fake에서 일부러 깨뜨려 본 뒤에야 테스트가 실제 장애를 잡기 시작했다.
