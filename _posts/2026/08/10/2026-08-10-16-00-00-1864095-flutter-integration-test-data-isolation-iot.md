---
layout: post
title: "Flutter integration_test 테스트 데이터 격리 - IoT 앱 재실행 flaky 해결"
description: "Flutter integration_test에서 이전 테스트의 MQTT·FCM·기기 상태가 남아 flaky해지는 문제를 테스트 데이터 seed와 reset 패턴으로 격리하는 방법을 정리했다."
date: 2026-08-10
tags: [Flutter, Dart, integration_test, IoT, MQTT, Firebase, CI/CD]
comments: true
share: true
---

![Flutter integration_test IoT 테스트 데이터 격리 구조](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 그림에서 봐야 할 부분은 하나의 스마트홈 앱을 여러 테스트가 그대로 공유하지 않고, 실행마다 독립된 상태 상자에서 시작하게 만드는 구조다.

Flutter integration_test에서 테스트가 가끔만 실패한다면 코드보다 데이터가 문제일 가능성이 크다. 첫 실행에서는 보일러 제어가 통과했는데 두 번째 실행부터 알림 개수가 달라지거나 MQTT 연결 상태가 이미 `connected`로 시작하는 식이다. 실제 IoT 앱에서 테스트가 읽는 상태는 보통 세 군데에 흩어져 있다.

| 상태 | 남아 있으면 생기는 문제 | 격리 방법 |
|---|---|---|
| 로컬 DB·SecureStorage | 이전 로그인·기기 선택이 재사용됨 | 테스트 전 삭제 후 seed |
| Fake MQTT·FCM 큐 | 이전 이벤트를 현재 테스트가 소비함 | 테스트 ID별 큐 생성 |
| Fake BLE·Controller | 연결 상태가 초기화되지 않음 | 새 인스턴스 주입 |

## 테스트마다 TestScope를 새로 만든다

처음에는 `setUpAll()`에서 Fake 서비스를 한 번 만들고 모든 테스트가 공유하게 했다. 실행 시간은 줄었지만, 테스트 순서가 결과를 결정하는 구조가 됐다. `setUp()`마다 새 Scope를 만들고, 테스트 종료 때 reset하는 편이 조금 느려도 원인을 추적하기 쉽다.

아래 코드는 실제 Firebase나 BLE 플러그인을 부르지 않고, 테스트 전용 의존성을 매번 새로 조립하는 예시다.

```dart
class TestScope {
  TestScope() : testId = 'it-${DateTime.now().microsecondsSinceEpoch}' {
    mqtt = FakeMqttService(queueId: testId);
    ble = FakeBleService();
    notifications = FakeNotificationService();
    storage = MemoryStorage();
  }

  final String testId;
  late final FakeMqttService mqtt;
  late final FakeBleService ble;
  late final FakeNotificationService notifications;
  late final MemoryStorage storage;

  Future<void> seed() async {
    await storage.write('spaceId', 'test-space-$testId');
    await storage.write('selectedDeviceId', 'boiler-sim-01');
    ble.addDevice(FakeDevice(id: 'boiler-sim-01', connected: false));
  }

  Future<void> reset() async {
    await mqtt.clearQueue();
    await notifications.clear();
    await storage.clear();
    ble.reset();
  }
}
```

테스트 본문은 앱을 띄우기 전에 `seed()`를 호출하고, 끝난 뒤 정리한다. `tearDown()`이 실행되도록 `try/finally`까지 넣어 두면 assertion 실패 때도 다음 테스트에 찌꺼기가 남지 않는다.

```dart
late TestScope scope;

setUp(() async {
  scope = TestScope();
  await scope.seed();
});

tearDown(() async {
  await scope.reset();
});

testWidgets('보일러 경보를 한 번만 표시한다', (tester) async {
  await pumpTestApp(tester, scope: scope);

  scope.mqtt.publish('home/alarm', {'code': 'E03'});
  await tester.pump(const Duration(milliseconds: 300));

  expect(find.text('보일러 경보'), findsOneWidget);
  expect(scope.notifications.sentCount, 1);
});
```

## 이벤트는 테스트 ID로 구분한다

큐를 비우는 것만으로 부족한 경우도 있다. 비동기 MQTT 콜백이 늦게 도착하면 이전 테스트에서 발행한 이벤트가 `clearQueue()` 이후 다시 들어올 수 있다. 그래서 Fake 서비스 안에 `queueId`를 두고 현재 Scope의 이벤트만 전달하도록 막았다.

```dart
class FakeMqttService {
  FakeMqttService({required this.queueId});

  final String queueId;
  final _events = <({String id, String topic, Map<String, dynamic> body})>[];

  void publish(String topic, Map<String, dynamic> body) {
    _events.add((id: queueId, topic: topic, body: body));
  }

  Stream<Map<String, dynamic>> listen(String topic) async* {
    for (final event in _events.where((event) =>
        event.id == queueId && event.topic == topic)) {
      yield event.body;
    }
  }

  Future<void> clearQueue() async => _events.clear();
}
```

여기서 중요한 건 테스트 코드 안에서 실제 `MqttClient`를 흉내 내는 게 아니다. 앱이 의존하는 `MqttService` 인터페이스만 구현하면 된다. 실제 연결·재연결은 [앱 생명주기에서 MQTT 재연결을 검증한 글]({% post_url 2026-08-09-16-00-00-1839405-flutter-integration-test-app-lifecycle-mqtt-reconnect-iot %})처럼 별도 시나리오에서 확인하고, 데이터 격리 테스트에서는 결정적인 Fake만 사용한다.

## CI에서 특히 조심할 점

로컬에서는 통과하는데 GitHub Actions에서만 실패한다면 테스트 간 상태보다 프로세스 간 공유를 의심해야 한다. 같은 에뮬레이터를 재사용하는 경우 앱 데이터가 남을 수 있으므로 테스트 시작 시 앱 저장소를 초기화하거나, 실행 옵션으로 고유한 flavor와 application ID를 사용한다.

| 증상 | 확인할 것 |
|---|---|
| 두 번째 테스트부터 로그인됨 | SecureStorage와 앱 데이터 삭제 |
| 알림 개수가 2배가 됨 | FCM Fake의 listener와 큐 reset |
| MQTT 첫 assertion이 간헐적 실패 | `pump` 시간보다 이벤트 seed 순서 |
| 병렬 실행 때만 실패 | 테스트별 DB 파일명·queueId 분리 |

정리하면 `setUpAll()` 공유 Fake 하나로 시작한 것이 가장 큰 실패 원인이었다. `setUp()`에서 Scope를 새로 만들고, 데이터는 seed하고, 이벤트는 테스트 ID로 묶고, `tearDown()`에서 비우면 재실행 순서에 덜 흔들린다. 실기기 테스트를 늘리기 전에 이 격리부터 잡아야 CI 결과를 믿을 수 있다.
