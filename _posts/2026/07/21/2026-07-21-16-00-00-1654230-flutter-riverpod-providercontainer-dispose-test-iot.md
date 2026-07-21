---
layout: post
title: "Flutter Riverpod ProviderContainer dispose 테스트 - BLE·MQTT 구독 정리 검증"
description: "Flutter Riverpod ProviderContainer를 dispose할 때 BLE 스캔과 MQTT 구독이 실제로 정리되는지 ref.onDispose와 ProviderObserver로 검증하는 테스트 방법을 IoT 앱 코드로 정리했다."
date: 2026-07-21
tags: [Flutter, Riverpod, BLE, MQTT, IoT, 테스트]
comments: true
share: true
---

![Flutter Riverpod ProviderContainer dispose와 IoT 리소스 정리](/images/2026-07-21-flutter-riverpod-providercontainer-dispose-test.png)

Flutter Riverpod에서 `ProviderContainer`를 dispose하는 테스트는 예외가 안 나는지만 보면 부족하다. BLE 스캔 Stream과 MQTT 구독이 실제로 끊겼는지까지 확인해야 화면을 열고 닫을 때 리소스가 남지 않는다. 처음엔 `container.dispose()` 한 줄이면 끝나는 줄 알았는데, 해보니 구독을 정리했다는 증거가 없었다.

## 화면은 닫혔는데 MQTT가 계속 온다

IoT 화면을 `autoDispose` Provider로 만들고 BLE와 MQTT 스트림을 연결했다. 화면을 나간 뒤에도 로그가 계속 찍혔다. Provider가 사라지는 것과 외부 서비스의 `StreamSubscription`이 취소되는 것은 별개였다.

| 대상 | dispose 전 | dispose 후 기대값 |
|---|---|---|
| BLE 스캔 | 기기 발견 이벤트 수신 | `cancel()` 호출 1회 |
| MQTT 구독 | 상태 토픽 수신 | unsubscribe 및 listener 종료 |
| Provider 상태 | `AsyncData` 또는 `AsyncLoading` | 더 이상 상태 변경 없음 |

Provider 내부에서 구독을 만들었다면 수명 종료 지점에 정리 코드를 등록한다. 테스트에서는 실제 패키지 대신 인터페이스와 Fake를 주입한다.

```dart
class FakeBleService implements BleService {
  int cancelCount = 0;
  final controller = StreamController<Device>.broadcast();

  @override
  Stream<Device> scan() => controller.stream;

  @override
  Future<void> cancelScan() async {
    cancelCount++;
  }
}

final bleProvider = Provider<BleService>((ref) => throw UnimplementedError());

final deviceProvider = StreamProvider.autoDispose<Device>((ref) {
  final ble = ref.watch(bleProvider);
  final subscription = ble.scan().listen((_) {});

  ref.onDispose(() {
    subscription.cancel();
    ble.cancelScan();
  });

  return ble.scan();
});
```

여기서 한 번 실수했다. `listen()`한 스트림과 `return`한 스트림이 달라 Fake 이벤트가 두 번 들어왔다. 구독을 직접 관리할 거면 같은 스트림만 사용해야 한다.

## ProviderContainer로 수명주기를 재현한다

테스트에서는 `ProviderContainer`에 Fake를 override하고, 화면이 붙은 상태를 `listen()`으로 재현한 뒤 dispose한다.

```dart
test('ProviderContainer dispose 시 BLE 구독을 정리한다', () async {
  final fakeBle = FakeBleService();
  final container = ProviderContainer(
    overrides: [bleProvider.overrideWithValue(fakeBle)],
  );
  addTearDown(container.dispose);

  final sub = container.listen(deviceProvider, (_, __) {});
  await container.read(deviceProvider.future);

  container.dispose();
  await pumpEventQueue();

  expect(fakeBle.cancelCount, 1);
  sub.close();
});
```

`dispose()` 뒤 구독 취소가 비동기로 마무리될 수 있어 바로 검증하지 않고 `pumpEventQueue()`로 이벤트 큐를 비운다. 이 한 줄이 없으면 로컬에서는 통과하고 CI에서만 실패하기 쉽다.

MQTT도 Fake의 `unsubscribeCount`와 `disconnectCount`를 세면 된다. `ref.keepAlive()`를 호출한 Provider는 화면 이탈이 아니라 container 자체를 dispose할 때 종료되므로 두 테스트를 분리해야 한다.

## ProviderObserver는 보조 증거로 쓴다

리소스 취소 여부는 Fake 서비스의 카운터가 가장 정확하다. `ProviderObserver`는 어떤 Provider가 dispose됐는지 확인하는 보조 도구로 쓴다. Provider는 dispose됐는데 네이티브 연결이 살아 있을 수도 있다.

```dart
class DisposeObserver extends ProviderObserver {
  final disposed = <ProviderBase<Object?>>[];

  @override
  void didDisposeProvider(
    ProviderObserverContext context,
    ProviderBase<Object?> provider,
  ) {
    disposed.add(provider);
  }
}
```

테스트 기준은 세 가지다.

- Provider가 dispose됐는가
- BLE·MQTT subscription의 cancel이 호출됐는가
- dispose 이후 늦게 도착한 이벤트가 상태를 다시 바꾸지 않는가

마지막 항목이 특히 중요하다. IoT 앱에서는 연결 종료 직전에 들어온 stale 응답이 화면 상태를 되살린다. `ProviderContainer dispose` 테스트를 리소스 카운터와 늦은 이벤트 검증까지 묶으면 화면 전환 때 쌓이는 연결을 초기에 잡을 수 있다.
