---
layout: post
title: "Flutter integration_test MQTT 장애 시뮬레이션 - IoT 재시도와 중복 ACK 검증"
description: "Flutter integration_test에서 MQTT 지연·연결 끊김·중복 ACK를 Fake 서비스로 재현하고, IoT 재시도와 중복 명령을 검증하는 실전 테스트 패턴을 정리했다."
date: 2026-08-15
tags: [Flutter, integration_test, MQTT, IoT, CI/CD, Riverpod]
comments: true
share: true
---

![Flutter integration_test MQTT 네트워크 장애 시뮬레이션](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

이 그림에서 볼 부분은 연결 자체보다 네트워크 상태를 통제하면서 테스트해야 한다는 점이다.

Flutter integration_test에서 MQTT ACK를 기다리는 테스트를 만들었는데, ACK가 늦게 오거나 연결이 중간에 끊기는 상황은 실제 브로커에 의존하면 재현하기 어렵다. 처음엔 CI에서 가끔 실패하는 테스트를 그냥 한 번 더 실행하면 된다고 생각했다. 해보니 원인은 재시도가 아니라 **실패 상황을 테스트에서 만들 수 없었던 것**이었다.

## 어떤 장애를 고정할 것인가

IoT 보일러 제어 기준으로 MQTT Fake에 세 가지 시나리오를 넣었다.

| 시나리오 | 검증할 동작 | 실패하기 쉬운 부분 |
|---|---|---|
| ACK 지연 | timeout 전까지 pending 유지 | `pumpAndSettle()`에 의존 |
| 연결 끊김 | 재연결 후 같은 commandId 재전송 | 명령 중복 발행 |
| 중복 ACK | 첫 ACK만 성공 처리 | 상태가 두 번 닫힘 |

여기서 `publish()`가 성공했다는 사실과 기기가 명령을 처리했다는 사실은 다르다. 테스트도 이 둘을 한 번에 성공으로 취급하면 안 된다.

## Fake MQTT 서비스에 장애 주입하기

테스트에서 지연과 중복 ACK를 직접 제어할 수 있도록 Fake를 만들었다. 실제 `mqtt5_client`를 실행하지 않고도 Repository가 어떤 순서로 동작하는지 확인하는 구조다.

```dart
class FakeMqttService implements MqttService {
  bool disconnectOnPublish = false;
  Duration ackDelay = Duration.zero;
  bool duplicateAck = false;
  final published = <String>[];
  final _ackController = StreamController<MqttAck>.broadcast();

  @override
  Stream<MqttAck> get acks => _ackController.stream;

  @override
  Future<void> publishCommand(String commandId, String payload) async {
    published.add(commandId);

    if (disconnectOnPublish) {
      throw MqttDisconnectedException();
    }

    await Future<void>.delayed(ackDelay);
    final ack = MqttAck(commandId: commandId, success: true);
    _ackController.add(ack);
    if (duplicateAck) _ackController.add(ack);
  }

  Future<void> dispose() => _ackController.close();
}
```

`disconnectOnPublish`를 `true`로 바꾸는 것만으로 연결 끊김을 만들 수 있다. 단, 재연결 로직이 같은 명령을 다시 보내는 구조라면 `published`에 저장된 ID가 정말 같은지 확인해야 한다. 새 UUID를 만들면 재시도처럼 보이지만 실제로는 다른 명령이 된다.

## integration_test에서 지연을 검증하는 방법

ACK 대기는 화면이 안정됐는지가 아니라 Controller 상태를 조건으로 기다려야 한다. 이번에는 테스트가 끝났을 때 호출 횟수와 최종 상태를 함께 확인했다.

```dart
testWidgets('MQTT ACK가 늦어도 한 번만 성공 처리한다', (tester) async {
  final mqtt = FakeMqttService()
    ..ackDelay = const Duration(milliseconds: 800)
    ..duplicateAck = true;
  final container = createTestContainer(mqtt);

  await tester.pumpWidget(buildTestApp(container));
  await tester.tap(find.byKey(const Key('powerButton')));
  await tester.pump(const Duration(milliseconds: 300));

  expect(find.text('처리 중'), findsOneWidget);
  await tester.pump(const Duration(milliseconds: 600));
  await tester.pump();

  expect(find.text('켜짐'), findsOneWidget);
  expect(mqtt.published, hasLength(1));
  expect(container.read(commandProvider).handledAckCount, 1);
  await mqtt.dispose();
});
```

중복 ACK 방어는 `commandId`별 처리 여부를 저장하는 쪽이 단순했다. Stream 구독 콜백에서 무조건 `state = success`를 실행하면 두 번째 ACK가 들어왔을 때 로그나 후속 이벤트가 중복된다. 실제 기기와 AWS IoT Core에서는 재전송이 충분히 생길 수 있으므로, 이 케이스는 통합 테스트에 남겨둘 가치가 있다.

## CI에서 특히 조심한 부분

Fake의 `StreamController`와 MQTT 구독을 `tearDown`에서 닫지 않으면 다음 테스트가 이전 ACK를 받는다. 그러면 단독 실행은 통과하고 전체 실행에서만 실패한다. 또 테스트마다 고정된 `commandId`를 쓰면 병렬 실행 시 로그가 섞이므로 실행 ID를 붙이되, **한 번의 재시도 안에서는 같은 ID를 유지**해야 한다.

```dart
addTearDown(() async {
  await mqtt.dispose();
  container.dispose();
});
```

솔직하게 정리하면, MQTT 장애 테스트의 목적은 네트워크를 실제처럼 느리게 만드는 데 있지 않다. 지연, 단절, 중복이라는 경계 조건에서 앱이 같은 명령을 두 번 처리하지 않는지 빠르게 확인하는 데 있다. `published`의 개수, `commandId`, 최종 화면 상태 세 가지를 같이 검사하면 flaky 테스트를 원인 있는 테스트로 바꿀 수 있다.
