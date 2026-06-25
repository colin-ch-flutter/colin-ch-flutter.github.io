---
layout: post
title: "mqtt5_client Unit Test - Fake 서비스로 MQTT 연결 로직 격리하기"
description: "mqtt5_client의 MqttClient는 생성자에 host/port를 받아 직접 Mock이 어렵다. abstract MqttService 인터페이스와 FakeMqttService 패턴으로 MQTT 연결·구독·발행 로직을 Unit Test하는 방법을 실제 코드로 정리했다."
date: 2026-06-24
tags: [Flutter, UnitTest, MQTT, mqtt5_client, Dart]
comments: true
share: true
---
![서버와 네트워크 연결 테스트](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

[이전 편에서 flutter_blue_plus의 static 메서드를 Wrapper로 격리했다.]({% post_url 2026-06-24-10-00-00-654285-flutter-blue-plus-ble-unit-test-wrapper-isolation %}) MQTT도 비슷한 벽이 있다. `mqtt5_client`의 `MqttClient`는 생성자에 host와 port를 바로 받는다. 추상 계층 없이 테스트에서 교체할 방법이 없다.

## MqttClient를 직접 Mock하면 생기는 문제

처음엔 `@GenerateMocks([MqttClient])`로 시작했다. 빌드는 된다. 그런데 문제가 누적된다.

`MqttClient`는 상태 기계(state machine)다. `connect()` 내부에서 콜백을 직접 호출하고, `connectionStatus`를 갱신하고, `updates` Stream을 관리한다. Mock으로 이 흐름을 흉내내려면 `when().thenAnswer()`를 10개 이상 세팅해야 한다.

`updates`의 타입이 `Stream<List<MqttReceivedMessage<MqttMessage>>>`. 이걸 stubbing하려면 타입 파라미터를 하나도 틀리지 않아야 한다. 한 번이라도 맞지 않으면 런타임에서 `type cast` 에러가 나는데 에러 메시지가 어디서 터지는지도 파악하기 어렵다.

결국 Mock 세팅 코드가 테스트 본 코드보다 길어진다. [MQTT 기초 편]({% post_url 2026-05-12-10-31-49-160485-mqtt-protocol-fundamentals-iot-heart %})에서 설계한 레이어 구조를 다시 꺼냈다.

## abstract MqttService 인터페이스 설계

Controller는 mqtt5_client를 직접 알지 못하게 한다. 얇은 추상 계층 하나만 바라본다.

```dart
// lib/data/services/mqtt_service.dart
abstract class MqttService {
  Future<bool> connect();
  Future<void> disconnect();
  void publish(String topic, String payload);
  Stream<String> subscribe(String topic);
  bool get isConnected;
}
```

`subscribe()`를 `Stream<String>`으로 단순화한 게 포인트다. `mqtt5_client`의 `MqttReceivedMessage` 타입을 그대로 노출하면 테스트에서도 그 타입을 생성해야 한다. payload 문자열만 내보내면 테스트 코드가 훨씬 단순해진다.

## 실제 구현체 — mqtt5_client 위임

```dart
// lib/data/services/real_mqtt_service.dart
class RealMqttService implements MqttService {
  late final MqttClient _client;
  final _controllers = <String, StreamController<String>>{};

  RealMqttService({
    required String host,
    required int port,
    required String clientId,
  }) {
    _client = MqttClient.withPort(host, clientId, port);
    _client.logging(on: false);
    _client.keepAlivePeriod = 20;
    _client.onDisconnected = _onDisconnected;
  }

  @override
  Future<bool> connect() async {
    try {
      await _client.connect();
      if (_client.connectionStatus?.state == MqttConnectionState.connected) {
        _listenUpdates();
        return true;
      }
      return false;
    } catch (_) {
      return false;
    }
  }

  @override
  void publish(String topic, String payload) {
    final builder = MqttClientPayloadBuilder()..addString(payload);
    _client.publishMessage(topic, MqttQos.atLeastOnce, builder.payload!);
  }

  @override
  Stream<String> subscribe(String topic) {
    _client.subscribe(topic, MqttQos.atLeastOnce);
    _controllers[topic] ??= StreamController<String>.broadcast();
    return _controllers[topic]!.stream;
  }

  @override
  bool get isConnected =>
      _client.connectionStatus?.state == MqttConnectionState.connected;

  @override
  Future<void> disconnect() async => _client.disconnect();

  void _listenUpdates() {
    _client.updates?.listen((messages) {
      for (final msg in messages) {
        final payload = MqttPublishPayload.bytesToStringAsString(
          (msg.payload as MqttPublishMessage).payload.message,
        );
        _controllers[msg.topic]?.add(payload);
      }
    });
  }

  void _onDisconnected() {}
}
```

## FakeMqttService — 테스트용 구현체

실제 브로커 없이 동작하는 Fake를 만든다. Mockito가 아니라 직접 구현이다.

```dart
// test/fakes/fake_mqtt_service.dart
class FakeMqttService implements MqttService {
  bool _connected = false;
  final _controllers = <String, StreamController<String>>{};
  final List<String> publishedTopics = [];
  final List<String> publishedPayloads = [];

  bool connectShouldFail = false;

  @override
  Future<bool> connect() async {
    if (connectShouldFail) return false;
    _connected = true;
    return true;
  }

  @override
  Future<void> disconnect() async {
    _connected = false;
    for (final c in _controllers.values) {
      await c.close();
    }
    _controllers.clear();
  }

  @override
  void publish(String topic, String payload) {
    publishedTopics.add(topic);
    publishedPayloads.add(payload);
  }

  @override
  Stream<String> subscribe(String topic) {
    _controllers[topic] ??= StreamController<String>.broadcast();
    return _controllers[topic]!.stream;
  }

  @override
  bool get isConnected => _connected;

  // 테스트에서 메시지 주입용
  void injectMessage(String topic, String payload) {
    _controllers[topic]?.add(payload);
  }
}
```

`injectMessage()`가 핵심이다. 테스트에서 특정 토픽으로 메시지를 밀어 넣을 수 있다. 실제 브로커 없이 수신 시나리오를 재현한다.

## Controller 테스트 작성

```dart
// test/controllers/boiler_controller_test.dart
void main() {
  late FakeMqttService fakeMqtt;
  late BoilerController controller;

  setUp(() {
    fakeMqtt = FakeMqttService();
    controller = BoilerController(mqttService: fakeMqtt);
  });

  tearDown(() async {
    await fakeMqtt.disconnect();
    controller.dispose();
  });

  test('연결 성공 시 isConnected가 true가 된다', () async {
    await controller.connect();
    expect(controller.isConnected, isTrue);
  });

  test('연결 실패 시 에러 상태가 된다', () async {
    fakeMqtt.connectShouldFail = true;
    await controller.connect();
    expect(controller.isConnected, isFalse);
    expect(controller.connectionError, isNotNull);
  });

  test('보일러 ON 명령 발행', () async {
    await controller.connect();
    controller.sendBoilerOn();
    expect(fakeMqtt.publishedTopics.last, 'home/boiler/command');
    expect(fakeMqtt.publishedPayloads.last, contains('"command":"on"'));
  });

  test('상태 메시지 수신 시 UI 상태 갱신', () async {
    await controller.connect();
    fakeMqtt.injectMessage(
      'home/boiler/status',
      '{"temperature": 45.2, "isOn": true}',
    );
    await Future.delayed(Duration.zero);
    expect(controller.currentTemperature, 45.2);
    expect(controller.isBoilerOn, isTrue);
  });
}
```

`Future.delayed(Duration.zero)`는 Stream 처리를 마이크로태스크 큐에서 소화할 시간을 준다. 이게 없으면 `injectMessage()` 직후 상태가 갱신되기 전에 `expect`가 실행돼서 테스트가 뻥 뚫린다.

![Fake 서비스 의존성 주입 구조 — Controller가 MqttService 인터페이스만 바라본다](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## 삽질한 부분

**subscribe() 타이밍**

Controller 생성자에서 즉시 `subscribe()`를 호출하면 `connect()` 전에 StreamController가 만들어진다. Fake에서는 괜찮지만 Real에서는 `_client.subscribe()`가 연결 전에 호출돼 패킷이 브로커에 도달하지 않는다. `connect()` 완료 콜백 안에서 subscribe하도록 순서를 강제했다.

**StreamController broadcast 안 붙이면**

처음에 `StreamController<String>()`로 만들었더니 "Bad state: Stream has already been listened to" 에러가 났다. Controller 내부에서 listen하고, 테스트에서도 listen하면 두 구독자가 생긴다. `.broadcast()`가 없으면 두 번째 listen에서 터진다.

**tearDown에서 disconnect() 필수**

`disconnect()`에서 `_controllers`를 close하지 않으면 tearDown 이후에도 stream이 살아있다. 다음 테스트에서 이전 테스트의 이벤트가 흘러들어와 이상한 실패가 나는데, 어느 테스트가 원인인지 추적하기가 꽤 고통스럽다.

---

다음 편은 GetX Repository 레이어의 오프라인 캐시 로직 테스트다. Realm 없이 FakeLocalDataSource만으로 캐시 히트·미스 시나리오를 검증하는 방법을 다룬다.

**참고 링크**
- [mqtt5_client pub.dev](https://pub.dev/packages/mqtt5_client)
- [Flutter Mockito 공식 가이드](https://docs.flutter.dev/cookbook/testing/unit/mocking)
- [mock-mqtt-client-test GitHub 예제](https://github.com/moritz89/mock-mqtt-client-test)
