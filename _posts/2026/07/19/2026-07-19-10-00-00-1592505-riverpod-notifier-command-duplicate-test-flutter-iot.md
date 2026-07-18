---
layout: post
title: "Flutter IoT Riverpod Notifier 테스트 - MQTT 명령 중복 실행과 AsyncValue 상태 전이 검증"
description: "Flutter IoT 스마트홈 앱에서 Riverpod Notifier의 MQTT 명령 중복 실행을 막고 AsyncValue loading·data·error 상태 전이를 테스트하는 방법을 정리했다."
date: 2026-07-19
tags: [Flutter, Dart, Riverpod, MQTT, mqtt5_client, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod Notifier 테스트와 MQTT 명령](https://images.unsplash.com/photo-1556742049-0cfed4f6a45d?w=1200&q=80)

Flutter IoT 스마트홈 앱에서 Riverpod Notifier 테스트는 값이 바뀌었는지만 보는 것으로 부족하다. MQTT 전원 명령을 보내는 순간 버튼을 두 번 누르면 명령이 두 번 발행되지 않는지, `loading → data` 또는 `loading → error` 전이가 정확한지까지 확인해야 실제 장애를 줄일 수 있다. 처음엔 `sendCommand()` 호출 횟수만 검사했는데, 해보니 화면은 성공으로 보이는데도 중복 명령이 나가는 경우를 놓치고 있었다.

## 테스트 대상은 상태와 부수효과로 나눈다

보일러 전원 토글을 담당하는 Notifier에 MQTT 서비스와 현재 상태 저장소를 주입한다고 가정했다. `isSending` 같은 별도 필드 대신 `AsyncValue`를 사용하면 화면은 상태 하나만 구독하면 된다.

| 검증 대상 | 성공 기준 | 놓치면 생기는 문제 |
| --- | --- | --- |
| 최초 상태 | `AsyncData(false)` | 화면 진입 때 버튼이 깜박임 |
| 명령 실행 중 | `AsyncLoading` | 연속 탭으로 MQTT 중복 발행 |
| 발행 성공 | `AsyncData(true)` | 실제 상태와 화면 표시 불일치 |
| 발행 실패 | `AsyncError` | 실패했는데 성공처럼 보임 |

핵심은 MQTT 응답이 도착할 때까지 기다리는 동안 같은 명령을 다시 실행하지 않는 것이다. `AsyncLoading`을 확인하는 가드가 없으면 네트워크가 느린 환경에서 재현된다.

## Fake MQTT 서비스로 실행 순서를 고정한다

실제 `mqtt5_client`를 테스트에 붙이면 브로커 연결 상태와 타이밍이 섞인다. 명령 발행 횟수와 성공·실패 시점을 직접 조절하는 Fake를 만들었다.

```dart
abstract interface class MqttCommandService {
  Future<void> publishPower({required String deviceId, required bool on});
}

class FakeMqttCommandService implements MqttCommandService {
  int publishCount = 0;
  bool shouldFail = false;
  final Completer<void> gate = Completer<void>();

  @override
  Future<void> publishPower({required String deviceId, required bool on}) async {
    publishCount++;
    await gate.future;
    if (shouldFail) throw StateError('MQTT publish failed');
  }
}
```

`gate`를 열기 전까지 Future가 끝나지 않게 한 이유는 두 번째 탭을 테스트하기 위해서다. 실제 브로커를 흉내 내는 Fake보다, 테스트가 필요한 시간만 멈출 수 있는 Fake가 훨씬 안정적이었다.

## Notifier에 중복 실행 가드를 둔다

명령이 끝나기 전에는 새 명령을 무시하고, 성공했을 때만 원하는 전원 상태를 반영한다. 실패하면 기존 상태를 유지한 채 에러를 전달한다.

```dart
class PowerNotifier extends AutoDisposeAsyncNotifier<bool> {
  late final MqttCommandService _mqtt;
  final String deviceId = 'boiler-a';
  bool _lastKnownOn = false;

  bool get lastKnownOn => _lastKnownOn;

  @override
  Future<bool> build() async {
    _mqtt = ref.read(mqttCommandServiceProvider);
    return _lastKnownOn;
  }

  Future<void> toggle() async {
    if (state.isLoading) return;

    final current = _lastKnownOn;
    state = const AsyncLoading();
    try {
      await _mqtt.publishPower(deviceId: deviceId, on: !current);
      _lastKnownOn = !current;
      state = AsyncData(_lastKnownOn);
    } catch (error, stackTrace) {
      state = AsyncError(error, stackTrace);
    }
  }
}
```

실제 코드에서는 `_mqtt`를 Provider에서 읽도록 구성한다. 여기서 중요한 건 `state.isLoading` 검사 위치다. MQTT 호출 뒤에 검사하면 이미 두 번 발행된 뒤라서 늦다.

## 두 번 눌러도 한 번만 발행되는지 검증한다

`ProviderContainer`에서 Notifier를 읽고, Future를 완료시키기 전 두 번 호출한다. 상태가 loading인 동안 두 번째 호출이 바로 반환되는지와 Fake의 발행 횟수를 함께 확인한다.

```dart
test('명령 처리 중 다시 눌러도 MQTT는 한 번만 발행한다', () async {
  final fake = FakeMqttCommandService();
  final container = ProviderContainer(
    overrides: [mqttCommandServiceProvider.overrideWithValue(fake)],
  );
  addTearDown(container.dispose);

  final notifier = container.read(powerProvider.notifier);
  await container.read(powerProvider.future);

  final first = notifier.toggle();
  await Future<void>.delayed(Duration.zero);
  expect(container.read(powerProvider), isA<AsyncLoading<bool>>());

  await notifier.toggle();
  expect(fake.publishCount, 1);

  fake.gate.complete();
  await first;
  expect(container.read(powerProvider).value, isTrue);
});
```

`Future.delayed(Duration.zero)`는 네트워크를 기다리는 코드가 아니다. 첫 번째 `toggle()`이 `publishPower()`를 호출하고 `gate`에서 멈춘 뒤, Notifier가 loading 상태를 반영할 짧은 이벤트 루프를 양보하는 용도다. 테스트가 환경에 따라 흔들리면 `container.listen`으로 상태 변화를 수집해 `AsyncLoading`을 확인하는 편이 낫다.

## 실패하면 기존 상태를 잃지 않는지 확인한다

MQTT 연결 끊김은 IoT 앱에서 흔하다. 실패 시 `AsyncError`가 되는지만 보지 말고, 직전 확인된 전원 상태까지 검증해야 재시도 버튼의 기준이 생긴다.

```dart
test('MQTT 발행 실패 시 에러와 기존 전원 상태를 남긴다', () async {
  final fake = FakeMqttCommandService()..shouldFail = true;
  final container = ProviderContainer(
    overrides: [mqttCommandServiceProvider.overrideWithValue(fake)],
  );
  addTearDown(container.dispose);

  final notifier = container.read(powerProvider.notifier);
  await container.read(powerProvider.future);
  fake.gate.complete();

  await notifier.toggle();

  final value = container.read(powerProvider);
  expect(value.hasError, isTrue);
  expect(value.error, isA<StateError>());
  expect(notifier.lastKnownOn, isFalse);
});
```

에러 상태의 이전 값을 `AsyncValue` 내부 필드에 무조건 의존하기보다 앱의 Riverpod 버전에 맞춰 마지막 성공 상태를 별도 모델로 보관하는 편이 안전하다. 내가 겪은 실수는 에러 상태를 만들면서 이전 값을 `null`로 덮어쓴 것이었다. 그 결과 화면은 실패했는데도 “꺼짐”과 “알 수 없음”을 구분하지 못했다.

## 짧게 정리하면

- Riverpod Notifier 테스트는 최종 값뿐 아니라 `loading → data/error` 전이를 검증한다.
- MQTT Fake는 Future 완료 시점을 제어해야 중복 탭과 타이밍 문제를 재현할 수 있다.
- `state.isLoading` 가드는 발행 호출 전에 둔다.
- 실패 시 `AsyncError`와 마지막 성공 상태를 함께 보존해야 재시도 UX가 흔들리지 않는다.

실제 브로커와 BLE 기기를 연결하는 테스트는 느리고 실패 원인도 넓다. 명령 횟수, 상태 전이, dispose 정리는 ProviderContainer 단위 테스트로 고정해두는 편이 Flutter IoT 앱 운영에 더 현실적인 안전망이었다.
