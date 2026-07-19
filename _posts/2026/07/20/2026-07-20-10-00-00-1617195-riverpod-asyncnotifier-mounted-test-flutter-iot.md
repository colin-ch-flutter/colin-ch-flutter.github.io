---
layout: post
title: "Flutter IoT Riverpod AsyncNotifier 테스트 - ref.mounted로 BLE stale 응답 막기"
description: "Flutter IoT 앱에서 Riverpod AsyncNotifier가 BLE·MQTT 응답을 기다리는 동안 dispose되는 문제를 재현하고 ref.mounted 가드와 Fake 서비스로 테스트하는 방법을 정리했다."
date: 2026-07-20
tags: [Flutter, Dart, Riverpod, BLE, MQTT, IoT, 테스트]
comments: true
share: true
---

![Flutter IoT Riverpod AsyncNotifier 테스트](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

이 그림에서 볼 부분은 기기 명령과 화면 생명주기가 서로 다른 속도로 끝난다는 점이다.

비동기 명령을 보낸 뒤 화면이 사라질 수 있다면 `AsyncNotifier` 안에서 응답을 바로 상태에 쓰면 안 된다. Flutter IoT 앱에서 보일러 전원을 끄고 바로 다른 Space로 이동했더니, 이전 기기의 MQTT 응답이 새 화면 상태에 섞이는 문제가 있었다. `ref.mounted`를 응답 직전에 확인하고, 테스트에서는 지연된 Fake 서비스와 `container.dispose()`로 이 상황을 재현하면 잡을 수 있다.

## 실제로 생긴 문제

처음엔 Repository가 결과를 반환하면 Notifier가 상태를 바꾸는 구조면 충분하다고 생각했다. 하지만 BLE 명령은 2초 타임아웃을 두었고, 사용자가 300ms 안에 화면을 나가면 Provider가 이미 dispose될 수 있다. MQTT도 비슷하다. 연결이 끊긴 뒤 늦게 도착한 ACK가 이미 다른 기기를 보고 있는 Controller에 반영됐다.

| 상황 | 기존 코드 | 가드 적용 후 |
| --- | --- | --- |
| 화면 유지 | 응답을 상태에 반영 | 응답을 상태에 반영 |
| 응답 전 화면 이탈 | dispose 뒤 상태 변경 시도 | 결과를 버리고 종료 |
| 기기 전환 중 늦은 ACK | 이전 요청 결과가 섞임 | 요청 세대가 다르면 무시 |

## `ref.mounted`를 응답 직전에 둔다

핵심은 `await` 앞에서 한 번 확인하는 것이 아니라, 비동기 작업이 끝난 직후 다시 확인하는 것이다.

```dart
class PowerNotifier extends AutoDisposeAsyncNotifier<bool> {
  @override
  Future<bool> build() async => false;

  Future<void> setPower(bool value) async {
    state = const AsyncLoading();

    try {
      final ok = await ref.read(deviceRepositoryProvider).setPower(value);

      // BLE/MQTT 응답을 기다리는 동안 Provider가 사라졌을 수 있다.
      if (!ref.mounted) return;

      state = ok
          ? AsyncData(value)
          : AsyncError(StateError('device rejected command'), StackTrace.current);
    } catch (error, stackTrace) {
      if (!ref.mounted) return;
      state = AsyncError(error, stackTrace);
    }
  }
}
```

이 가드는 예외를 숨기는 장치가 아니다. 이미 화면의 소유권이 끝난 Provider가 늦은 결과를 소비하지 않게 하는 수명 확인이다. 기기를 바꾸는 화면이라면 `mounted`만으로 부족할 때도 있다. 그때는 요청 번호를 함께 둔다.

```dart
int _requestId = 0;

Future<void> setPower(bool value) async {
  final requestId = ++_requestId;
  state = const AsyncLoading();
  final ok = await ref.read(deviceRepositoryProvider).setPower(value);

  if (!ref.mounted || requestId != _requestId) return;
  state = AsyncData(ok && value);
}
```

## 지연된 Fake로 dispose를 재현한다

실제 BLE 기기를 테스트에 연결하면 타이밍이 매번 달라진다. 그래서 완료 시점을 직접 조절하는 Fake를 만들었다.

```dart
class DelayedDeviceRepository implements DeviceRepository {
  final completer = Completer<bool>();

  @override
  Future<bool> setPower(bool value) => completer.future;
}

test('Provider가 dispose된 뒤 늦은 응답은 상태를 바꾸지 않는다', () async {
  final fake = DelayedDeviceRepository();
  final container = ProviderContainer(overrides: [
    deviceRepositoryProvider.overrideWithValue(fake),
  });
  addTearDown(container.dispose);

  final sub = container.listen(powerNotifierProvider, (_, __) {});
  final notifier = container.read(powerNotifierProvider.notifier);
  final command = notifier.setPower(true);

  container.dispose();
  fake.completer.complete(true);
  await command;

  sub.close();
  expect(container.exists(powerNotifierProvider), isFalse);
});
```

여기서 흔히 놓치는 부분은 `container.dispose()`만 하고 Fake의 Future를 끝내지 않는 것이다. 그러면 테스트가 조용히 멈추거나 타임아웃이 난다. `Completer`를 반드시 완료시키고, `StreamSubscription`과 `ProviderContainer`도 정리해야 한다.

## 짧게 정리하면

- `await` 뒤에는 `ref.mounted`를 확인한다.
- 기기 전환처럼 순서가 중요하면 요청 번호를 함께 검증한다.
- 실제 BLE 대신 완료 시점을 제어하는 Fake를 사용한다.
- dispose 테스트에서도 Future와 listener를 끝까지 정리한다.

Flutter BLE와 MQTT의 연결 성공 여부만 테스트하면 이런 버그는 놓치기 쉽다. 응답이 늦게 오는 실패 상황까지 고정해야 화면 전환 뒤의 유령 상태를 줄일 수 있다.
