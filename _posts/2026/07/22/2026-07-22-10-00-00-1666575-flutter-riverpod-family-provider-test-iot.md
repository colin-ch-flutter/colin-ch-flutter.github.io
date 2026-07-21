---
layout: post
title: "Flutter Riverpod family provider 테스트 - IoT 공간별 상태 격리"
description: "Flutter Riverpod family provider로 거실·침실·보일러 상태를 분리하고, ProviderContainer override와 Fake MQTT를 이용해 공간별 IoT 상태가 섞이지 않는지 테스트하는 방법을 정리했다."
date: 2026-07-22
tags: [Flutter, Dart, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter Riverpod family provider 테스트와 IoT 공간별 상태 격리](/images/2026-07-22-flutter-riverpod-family-provider-test.png)

Flutter Riverpod family provider 테스트의 핵심은 `roomId`별로 독립된 `ProviderContainer`를 만들고, MQTT 서비스를 Fake로 교체하는 데 있다. 처음엔 하나의 테스트 컨테이너에서 거실과 침실을 번갈아 검증했는데, 이전 상태가 남아 테스트가 순서에 따라 달라졌다. family provider는 편리하지만, 테스트 격리를 직접 보장하지 않으면 IoT 앱에서 공간 상태가 섞이는 버그를 놓치기 쉽다.

## 왜 family provider를 따로 검증해야 하나

스마트홈 앱에는 같은 타입의 기기가 여러 공간에 있다. `boilerProvider('living-room')`과 `boilerProvider('bedroom')`은 선언을 공유하지만 상태와 MQTT 토픽은 달라야 한다.

| 검증 대상 | 기대 결과 | 놓치기 쉬운 실패 |
|---|---|---|
| 거실 온도 변경 | 거실 상태만 변경 | 침실 카드도 다시 빌드됨 |
| 침실 MQTT 수신 | 침실 provider만 갱신 | 마지막 수신 값이 모든 공간에 반영됨 |
| 컨테이너 dispose | Stream 구독 해제 | 테스트 뒤에도 메시지 수신 |

`ref.watch`가 family 인자를 빠뜨리거나 Repository가 싱글턴 상태를 사용하면 화면에서는 정상처럼 보일 수 있다. 두 공간을 동시에 올려야 이 문제가 드러난다.

## Fake MQTT를 family provider에 주입하기

실제 브로커 대신 토픽별 메시지를 흘려보낼 수 있는 Fake 서비스를 주입한다.

```dart
class FakeMqttService implements MqttService {
  final _messages = StreamController<MqttMessage>.broadcast();

  void publish(String topic, Map<String, dynamic> payload) {
    _messages.add(MqttMessage(topic, payload));
  }

  @override
  Stream<MqttMessage> messagesFor(String topic) =>
      _messages.stream.where((message) => message.topic == topic);

  @override
  Future<void> dispose() => _messages.close();
}

final boilerProvider =
    StreamNotifierProvider.family<BoilerNotifier, BoilerState, String>(
  BoilerNotifier.new,
);

class BoilerNotifier extends FamilyStreamNotifier<BoilerState, String> {
  @override
  Stream<BoilerState> build(String roomId) {
    final topic = 'homes/demo/$roomId/boiler/state';
    return ref.watch(mqttServiceProvider).messagesFor(topic).map(
          (message) => BoilerState.fromJson(message.payload),
        );
  }
}
```

핵심은 `topic`을 `roomId`로 만드는 지점이다. 현재 공간을 전역 변수로 읽으면 family를 쓰는 의미가 사라진다.

## 두 공간을 동시에 올리는 테스트

이제 하나의 컨테이너에 Fake 서비스를 override하고, 두 provider가 서로 영향을 주지 않는지 확인한다.

```dart
test('공간별 boiler 상태는 서로 섞이지 않는다', () async {
  final mqtt = FakeMqttService();
  final container = ProviderContainer(
    overrides: [mqttServiceProvider.overrideWithValue(mqtt)],
  );
  addTearDown(() async {
    container.dispose();
    await mqtt.dispose();
  });

  final living = container.listen(boilerProvider('living-room'), (_, __) {});
  final bedroom = container.listen(boilerProvider('bedroom'), (_, __) {});

  mqtt.publish('homes/demo/living-room/boiler/state', {'temperature': 23});
  await Future<void>.delayed(Duration.zero);
  expect(container.read(boilerProvider('living-room')).temperature, 23);
  expect(container.read(boilerProvider('bedroom')).temperature, isNot(23));

  living.close();
  bedroom.close();
});
```

`listen`을 호출한 이유는 자동 dispose provider를 테스트 중 활성 상태로 유지하기 위해서다. `Future.delayed(Duration.zero)`는 Fake stream 이벤트가 이벤트 루프를 한 번 지난 뒤 상태에 반영되도록 기다린다. 재연결 타이머까지 있으면 `fakeAsync`로 시간을 제어하는 편이 낫다.

## 실제로 걸렸던 함정

처음엔 모든 메시지를 하나의 `StreamController`로 보내고 Notifier가 `where`로 토픽을 걸렀다. 기능은 맞았지만 테스트 종료 후 `StreamController was disposed` 로그가 반복됐다. `ProviderContainer.dispose()`와 서비스 `dispose()` 순서가 뒤섞였기 때문이다.

정리 순서는 테스트마다 고정하는 게 안전하다.

1. provider listener를 닫는다.
2. `ProviderContainer`를 dispose한다.
3. Fake MQTT stream을 닫는다.

실제 앱에서 MQTT 서비스를 전역 공유한다면 컨테이너마다 서비스를 닫으면 안 된다. [ProviderContainer dispose 테스트]({% post_url 2026-07-21-16-00-00-1654230-flutter-riverpod-providercontainer-dispose-test-iot %})처럼 구독 정리도 별도 검증하는 게 좋다.

## 짧게 정리하면

Flutter Riverpod family provider는 `roomId`를 상태 경계로 삼을 때 효과가 크다. 테스트에서는 두 개 이상의 family 인스턴스를 동시에 만들고, 토픽을 하나만 발행한 뒤 나머지 공간이 변하지 않는지 확인해야 한다. `ProviderContainer`와 Fake MQTT의 dispose 순서까지 고정하면 테스트 순서에 따른 간헐 실패도 줄어든다.
