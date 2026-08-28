---
layout: post
title: "Flutter integration_test 오프라인 명령 큐 - MQTT 재연결 후 중복 publish 검증"
description: "Flutter integration_test로 네트워크 단절 중 쌓인 스마트홈 명령이 MQTT 재연결 뒤 한 번만 publish되는지 검증하고, 중복 실행을 막는 테스트 구조를 정리했다."
date: 2026-08-28
tags: [Flutter, Dart, integration_test, MQTT, mqtt5_client, IoT, 스마트홈]
comments: true
share: true
---

![Flutter integration_test MQTT 오프라인 명령 큐 검증](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=1200&q=80)
*이 그림에서 볼 것은 네트워크가 끊겨도 사용자의 명령과 실제 기기 반영 시점을 분리하는 구조다.*

Flutter integration_test에서 버튼이 눌렸다는 사실만 확인하면 MQTT 오프라인 큐의 버그를 놓친다. 스마트홈 앱은 네트워크가 끊긴 순간에도 사용자의 마지막 명령을 저장할 수 있는데, 재연결 때 같은 명령이 두 번 publish되면 보일러나 조명이 예상과 다르게 움직인다. 화면 조작, 연결 복구, 큐 비우기를 한 시나리오로 묶어 검증해야 한다.

## 단위 테스트만으로 부족했던 이유

`fakeAsync`로 재연결 타이머와 큐 처리 순서를 검증하는 테스트는 이미 있었다. 그런데 실제 화면에서는 제어 버튼이 `pending` 상태가 되고, 앱이 복귀하면서 세션 Provider가 다시 만들어진다. 이때 `flush()`가 두 번 호출되는 문제는 Repository 단위 테스트에서 재현되지 않았다.

내가 확인한 실패 흐름은 이랬다.

| 시점 | 화면 상태 | MQTT 큐 |
| --- | --- | --- |
| 네트워크 단절 | 명령 전송 중 | `set_power_on` 1개 |
| 재연결 시작 | 재시도 표시 | 1개 유지 |
| 세션 복구 완료 | 성공 표시 | 1회 publish 후 비움 |

핵심은 재연결 이벤트와 화면의 `resumed` 처리를 각각 큐 flush의 트리거로 쓰지 않는 것이다. 둘 다 실행되더라도 `flushInProgress` 같은 잠금으로 한 번만 처리해야 한다.

## 테스트 전용 MQTT Fake

실제 broker에 연결하지 않고도 publish 횟수와 payload를 기록하는 Fake를 주입한다. 이 코드는 테스트 앱에서 네트워크 단절과 복구를 버튼으로 제어할 수 있게 만든 최소 예시다.

```dart
class RecordingMqtt implements MqttGateway {
  bool connected = true;
  final published = <String>[];

  @override
  Future<void> publish(String topic, String payload) async {
    if (!connected) throw const MqttDisconnectedException();
    published.add('$topic:$payload');
  }

  Future<void> goOffline() async => connected = false;

  Future<void> goOnline() async => connected = true;
}
```

테스트에서는 실제 사용자처럼 화면에서 전원 버튼을 누른다. 연결을 끊은 뒤 버튼을 다시 누르면 명령이 큐에 들어가고, `reconnect` 버튼을 누른 뒤에는 기록된 publish가 정확히 한 건인지 확인한다.

```dart
testWidgets('MQTT 재연결 후 오프라인 명령을 한 번만 전송한다', (tester) async {
  final mqtt = RecordingMqtt();
  await launchTestApp(mqtt: mqtt);

  await tester.tap(find.byKey(const Key('network-offline')));
  await tester.tap(find.byKey(const Key('boiler-power-on')));
  await tester.pump();

  expect(find.text('전송 대기 중'), findsOneWidget);
  expect(mqtt.published, isEmpty);

  await tester.tap(find.byKey(const Key('network-online')));
  await tester.tap(find.byKey(const Key('mqtt-reconnect')));
  await tester.pumpAndSettle();

  expect(mqtt.published, hasLength(1));
  expect(mqtt.published.single, contains('set_power_on'));
  expect(find.text('전송 완료'), findsOneWidget);
});
```

여기서 `pumpAndSettle()`은 연결 복구와 큐 flush가 끝나는 지점에만 사용했다. 모든 단계에 넣으면 중복 호출을 숨길 수 있다. 특히 재연결 버튼을 두 번 빠르게 탭했을 때도 publish가 한 건인지 별도 케이스로 검사해야 한다.

## 주의할 점

큐에 저장하는 명령에는 `commandId`를 붙이는 편이 안전하다. 같은 payload라도 사용자가 두 번 누른 명령은 서로 다른 요청일 수 있고, 재연결 과정에서 같은 이벤트가 다시 전달될 수도 있다. 반대로 전원 토글처럼 최신 상태만 의미 있는 명령은 큐에 쌓기보다 같은 기기의 이전 명령을 교체하는 정책이 맞을 수 있다.

또한 integration_test가 production MQTT 주소를 읽지 않도록 테스트 진입점에서 환경값을 강제해야 한다. Fake를 주입했다고 생각했는데 초기화 코드가 먼저 실행돼 실제 broker 연결이 발생하면, 테스트는 느려지는 수준이 아니라 운영 기기를 움직이는 사고가 된다.

짧게 정리하면 `Flutter integration_test`의 MQTT 큐 검증은 버튼 성공 여부가 끝이 아니다. 네트워크 단절 중 명령이 보존되는지, 재연결 뒤 정확히 한 번 전송되는지, 전송 후 큐가 비워지는지를 같은 시나리오에서 확인해야 한다.
