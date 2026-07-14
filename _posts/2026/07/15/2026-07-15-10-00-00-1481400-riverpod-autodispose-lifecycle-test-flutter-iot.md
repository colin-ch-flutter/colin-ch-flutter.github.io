---
layout: post
title: "Flutter IoT Riverpod autoDispose 테스트 - BLE 스트림 누수 막는 생명주기 검증"
description: "Flutter IoT 앱에서 Riverpod autoDispose와 ref.onDispose로 BLE 스트림을 정리하고, ProviderContainer 테스트로 화면 이탈 뒤 구독 누수를 검증하는 방법을 정리했다."
date: 2026-07-15
tags: [Flutter, Riverpod, Dart, BLE, IoT, flutter_blue_plus]
comments: true
share: true
---

![Flutter IoT 앱의 BLE 스트림 생명주기 테스트](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Flutter IoT 앱에서 Riverpod `autoDispose`를 붙였다고 BLE 구독이 자동으로 안전하게 끝나는 건 아니다. 화면을 나간 뒤에도 `StreamSubscription`이 살아 있어 같은 기기 이벤트가 두 번씩 들어오는 문제가 생겼다. `ref.onDispose`에서 구독을 취소하고, `ProviderContainer` 테스트로 정리 호출을 확인하는 게 해결책이었다.

## 화면보다 스트림의 생명주기가 문제다

처음에는 `StreamProvider.autoDispose.family`면 충분하다고 생각했다.

| 상황 | 기대 | 놓치기 쉬운 부분 |
|---|---|---|
| 기기 화면 진입 | BLE 상태 구독 시작 | 네이티브 구독도 함께 시작됨 |
| 화면 뒤로 가기 | Provider 정리 | 직접 만든 구독은 남을 수 있음 |
| 같은 화면 재진입 | 이벤트 1회 수신 | 이전 구독까지 있으면 2회 수신 |

`flutter_blue_plus`의 characteristic notification을 Repository에서 변환하는 구조라면 Provider가 사라지는 것과 BLE 구독 취소는 별개다. Provider가 소유한 기기별 구독을 `ref.onDispose`로 묶어야 한다.

## ref.onDispose에서 BLE 구독을 끊는다

아래처럼 원본 구독과 중간 `StreamController`를 같은 정리 콜백에 등록한다.

```dart
final deviceStateProvider = StreamProvider.autoDispose
    .family<DeviceState, String>((ref, deviceId) {
  final repository = ref.watch(bleRepositoryProvider);
  final output = StreamController<DeviceState>();
  final subscription = repository.watchState(deviceId).listen(
    output.add,
    onError: output.addError,
  );

  ref.onDispose(() {
    subscription.cancel();
    output.close();
    repository.stopWatching(deviceId);
  });
  return output.stream;
});
```

`StreamSubscription.cancel()`은 Future지만 `ref.onDispose` 콜백에서 완료까지 기다리지는 못한다. 취소 완료가 반드시 필요한 네이티브 SDK라면 Repository에 별도의 비동기 dispose 순서를 두는 편이 낫다. 반대로 전역 BLE 매니저까지 여기서 종료하면 다른 기기 화면의 연결이 끊길 수 있으니, 기기별 구독만 정리해야 한다.

## ProviderContainer로 화면 이탈을 재현한다

위젯 대신 listener를 닫아 마지막 구독자가 사라지는 상황을 만든다. Fake Repository에는 `stopWatching` 횟수와 활성 기기 목록을 남긴다.

```dart
test('마지막 listener가 사라지면 BLE 구독을 정리한다', () async {
  final fake = FakeBleRepository();
  final container = ProviderContainer(overrides: [
    bleRepositoryProvider.overrideWithValue(fake),
  ]);
  addTearDown(container.dispose);

  final listener = container.listen(
    deviceStateProvider('boiler-01'), (_, __) {}, fireImmediately: true,
  );
  fake.emit(const DeviceState(temperature: 42));
  await Future<void>.delayed(Duration.zero);
  expect(container.read(deviceStateProvider('boiler-01')).value?.temperature, 42);

  listener.close();
  await Future<void>.delayed(Duration.zero);
  expect(fake.stopWatchingCount, 1);
  expect(fake.activeDeviceIds, isEmpty);
});
```

`listener.close()` 직후 바로 검증하면 간헐적으로 실패할 수 있다. autoDispose 정리는 현재 이벤트 루프가 끝난 뒤 실행되므로 `Future.delayed(Duration.zero)`로 한 번 넘겼다. 이 테스트에서는 `pumpAndSettle`보다 이벤트 루프를 직접 넘기는 방식이 의도를 잘 보여준다.

## 구독 개수를 테스트 기준으로 둔다

```text
listener 생성 → BLE 구독 시작 → 상태 이벤트 수신
       ↓
listener close → autoDispose → ref.onDispose 실행
       ↓
구독 취소 + controller close + Repository 정리
```

같은 기기에 재진입해도 활성 구독이 1개인지, 화면을 나간 뒤 `stopWatching`이 1번만 호출되는지, 늦게 도착한 이벤트가 새 화면을 오염시키지 않는지를 확인한다. 상태 값만 검증하면 운영에서 발생하는 중복 이벤트를 놓친다.

Riverpod `autoDispose`는 Provider 수명을 줄여주지만 Repository가 만든 BLE 구독까지 추측해 취소하지는 않는다. `ref.onDispose`에 정리를 명시하고 `ProviderContainer`에서 listener 종료를 테스트하면 화면 재진입 때의 중복 이벤트와 스트림 누수를 잡을 수 있다. [ProviderContainer로 상태 전이를 검증한 패턴]({% post_url 2026-07-01-16-00-00-1049325-riverpod-providercontainer-test-flutter-iot %})과 함께 사용하면 위젯 없이도 생명주기를 확인할 수 있다.
