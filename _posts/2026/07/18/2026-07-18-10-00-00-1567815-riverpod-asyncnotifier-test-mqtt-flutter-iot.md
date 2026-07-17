---
layout: post
title: "Flutter IoT Riverpod AsyncNotifier 테스트 - MQTT 연결 끊김과 재시도 검증"
description: "Flutter IoT 앱의 Riverpod AsyncNotifier에서 MQTT 연결 중·성공·실패 상태와 재시도 횟수를 테스트하는 방법을 Fake Repository와 ProviderContainer로 정리했다."
date: 2026-07-18
tags: [Flutter, Dart, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod AsyncNotifier 테스트와 MQTT 연결 재시도 흐름](/images/2026-07-18-riverpod-asyncnotifier-test-mqtt.png)

그림에서 볼 부분은 MQTT 연결 상태를 화면에서 직접 기다리는 게 아니라 `AsyncNotifier`의 상태 전이와 재시도 경로로 검증한다는 점이다.

Flutter IoT 앱에서 Riverpod `AsyncNotifier`를 테스트할 때는 최종값 하나보다 `loading → data → error` 순서를 확인하는 편이 안전하다. 처음엔 MQTT Fake가 `connect()`만 성공하면 테스트가 끝난다고 생각했다. 해보니 연결 끊김 뒤 재시도 버튼이 실제로 한 번만 실행되는지, 실패 원인이 `AsyncError`로 전달되는지까지 확인해야 UI가 거짓 성공을 보여주지 않았다.

## Repository와 Notifier를 분리한다

실제 `mqtt5_client`를 테스트에 넣으면 브로커 주소, 네트워크, keep alive 타이밍이 따라온다. CI에서는 이 조건을 재현하기 어렵기 때문에 연결 책임을 Repository로 감싸고 Fake로 바꿨다.

```dart
abstract interface class MqttRepository {
  Future<void> connect();
  Future<void> disconnect();
}

class FakeMqttRepository implements MqttRepository {
  int connectCount = 0;
  bool shouldFail = false;

  @override
  Future<void> connect() async {
    connectCount++;
    if (shouldFail) throw Exception('broker unavailable');
  }

  @override
  Future<void> disconnect() async {}
}
```

`connectCount`는 단순한 상태값이 아니라 재시도 정책을 검증하는 Spy 역할도 한다. Fake 안에서 예외를 삼키면 Notifier는 성공 상태로 남기 때문에, 실패 조건에서는 반드시 throw해야 한다.

## ProviderContainer에서 상태 전이를 읽는다

아래 코드는 MQTT 연결 상태와 재시도 명령을 하나의 `AsyncNotifier`에 두되, 테스트에서는 Repository만 override하는 형태다. 코드 바로 아래 테스트에서 상태 순서와 호출 횟수를 함께 검증한다.

```dart
final mqttRepositoryProvider = Provider<MqttRepository>((ref) {
  throw UnimplementedError();
});

final mqttStatusProvider = AsyncNotifierProvider<MqttStatusNotifier, bool>(
  MqttStatusNotifier.new,
);

class MqttStatusNotifier extends AsyncNotifier<bool> {
  @override
  Future<bool> build() async {
    await ref.read(mqttRepositoryProvider).connect();
    return true;
  }

  Future<void> retry() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      await ref.read(mqttRepositoryProvider).connect();
      return true;
    });
  }
}

test('MQTT 연결 실패 후 retry가 한 번 실행된다', () async {
  final fake = FakeMqttRepository()..shouldFail = true;
  final container = ProviderContainer(
    overrides: [mqttRepositoryProvider.overrideWithValue(fake)],
  );
  addTearDown(container.dispose);

  final states = <AsyncValue<bool>>[];
  final sub = container.listen(
    mqttStatusProvider,
    (_, next) => states.add(next),
    fireImmediately: true,
  );
  addTearDown(sub.close);

  await container.read(mqttStatusProvider.future).catchError((_) => false);
  expect(container.read(mqttStatusProvider), isA<AsyncError<bool>>());

  fake.shouldFail = false;
  await container.read(mqttStatusProvider.notifier).retry();

  expect(fake.connectCount, 2);
  expect(container.read(mqttStatusProvider).value, true);
  expect(states.any((value) => value is AsyncLoading<bool>), isTrue);
});
```

여기서 헷갈렸던 건 `container.read(provider.future)`가 실패하면 테스트도 즉시 실패한다는 점이다. 의도적으로 실패를 검증하는 경우에는 `catchError`로 Future 예외를 소비하고, 최종 상태는 Provider 자체에서 `AsyncError`인지 확인해야 한다. 테스트마다 `ProviderContainer`와 listener를 닫지 않으면 이전 MQTT 이벤트가 다음 테스트에 섞인다.

## 상태별로 확인할 기준

| 상황 | 확인할 값 | 실패하기 쉬운 가정 |
|---|---|---|
| 최초 연결 | `AsyncLoading → AsyncData` | build가 동기라고 생각함 |
| 브로커 장애 | `AsyncError`와 예외 내용 | Repository가 예외를 삼킴 |
| 재시도 성공 | 호출 횟수와 `AsyncData` | 버튼 탭마다 무제한 connect |
| 테스트 종료 | listener·container 정리 | 스트림과 타이머가 계속 실행 |

솔직하게 정리하면 `AsyncNotifier` 테스트의 핵심은 Notifier를 직접 `new`하는 게 아니다. `ProviderContainer`에서 실제 의존성 그래프를 만들고, 외부 네트워크만 Fake로 치환해야 앱에서의 동작과 테스트가 같은 구조가 된다. MQTT 연결 끊김은 재현이 어렵지만, 실패 플래그와 호출 횟수를 고정하면 CI에서도 재시도 정책을 꾸준히 검증할 수 있다.
