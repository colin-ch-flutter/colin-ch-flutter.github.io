---
layout: post
title: "Flutter IoT Riverpod family 테스트 - 기기별 상태 오염 막기"
description: "Flutter IoT 스마트홈 앱에서 Riverpod family Provider를 기기 ID별로 격리하고, BLE와 MQTT 상태가 다른 기기로 섞이지 않는 테스트 방법을 정리했다."
date: 2026-07-15
tags: [Flutter, Riverpod, Dart, BLE, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT 스마트홈 앱의 기기별 상태 테스트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter IoT 스마트홈 앱에서 Riverpod `family` Provider를 쓰면 기기별 상태를 나누기 쉽다. 문제는 테스트에서 같은 `ProviderContainer`를 재사용하거나 잘못된 override를 넣을 때다. 보일러 A의 MQTT 응답이 보일러 B 카드에 표시되는 식의 상태 오염은 실제 기기보다 테스트에서 먼저 잡아야 한다.

## 기기 ID가 Provider의 경계가 된다

처음에는 모든 기기 상태를 하나의 `Map<String, DeviceState>`로 관리했다. 화면은 빨리 만들 수 있었지만, BLE 등록 화면에서 들어온 임시 상태와 MQTT로 받은 운영 상태가 한 Map에서 섞였다. 한 기기의 갱신이 다른 카드까지 다시 그리게 되는 문제도 있었다.

`family`를 사용하면 Provider 인자 자체가 캐시 키가 된다.

| 구분 | 잘못 나눈 기준 | 기기별 family 기준 |
|---|---|---|
| 상태 키 | 전역 Map의 문자열 | `deviceId` |
| 테스트 | 모든 기기 상태를 한 번에 확인 | 기기별 상태를 따로 확인 |
| 실패 원인 | 어떤 갱신이 섞였는지 추적 어려움 | A 또는 B의 입력으로 좁혀짐 |

이전 [ProviderContainer 상태 전이 테스트]({% post_url 2026-07-01-16-00-00-1049325-riverpod-providercontainer-test-flutter-iot %})처럼 테스트 루트를 만들되, 이번에는 `deviceStateProvider('boiler-a')`와 `deviceStateProvider('boiler-b')`를 동시에 읽는다.

## family Provider는 Repository 호출까지 ID를 전달한다

아래 구조는 실제 BLE 스캔이나 MQTT 브로커에 연결하지 않고, Repository가 받은 `deviceId`를 올바르게 사용했는지만 확인하기 위한 최소 예제다.

```dart
final deviceStateProvider =
    FutureProvider.autoDispose.family<DeviceState, String>((ref, deviceId) {
  final repository = ref.watch(deviceRepositoryProvider);
  return repository.fetchState(deviceId);
});

class FakeDeviceRepository implements DeviceRepository {
  final calls = <String>[];
  final states = <String, DeviceState>{
    'boiler-a': DeviceState(isOn: true, temperature: 22),
    'boiler-b': DeviceState(isOn: false, temperature: 18),
  };

  @override
  Future<DeviceState> fetchState(String deviceId) async {
    calls.add(deviceId);
    return states[deviceId]!;
  }
}
```

핵심은 Fake가 고정된 상태를 반환하는 것보다 호출된 ID를 기록하는 것이다. 화면에 값이 맞게 보이는 것만 검사하면 Provider 내부에서 다른 ID를 넘기는 실수를 놓칠 수 있다.

## 두 기기를 한 컨테이너에서 읽어본다

기기별 캐시 키가 분리됐는지 확인하려면 두 family 인스턴스를 같은 컨테이너에서 읽고, 각각의 결과와 Repository 호출 순서를 검증한다.

```dart
test('기기별 family 상태가 서로 섞이지 않는다', () async {
  final fake = FakeDeviceRepository();
  final container = ProviderContainer(
    overrides: [deviceRepositoryProvider.overrideWithValue(fake)],
  );
  addTearDown(container.dispose);

  final boilerA = await container.read(
    deviceStateProvider('boiler-a').future,
  );
  final boilerB = await container.read(
    deviceStateProvider('boiler-b').future,
  );

  expect(boilerA.isOn, isTrue);
  expect(boilerA.temperature, 22);
  expect(boilerB.isOn, isFalse);
  expect(boilerB.temperature, 18);
  expect(fake.calls, ['boiler-a', 'boiler-b']);
});
```

처음엔 `container.read(deviceStateProvider('boiler-a'))`만 하고 내부 Map을 검사했다. 그 테스트는 통과했지만 B 카드를 추가하자 문제가 드러나지 않았다. family 테스트는 최소 두 개의 ID를 넣어야 의미가 있다. 특히 `deviceId`를 생략한 Repository 호출, 임시 문자열을 하드코딩한 경우를 바로 잡아낸다.

## 같은 ID의 재조회와 다른 ID의 재조회를 구분한다

`family`는 인자가 같으면 같은 Provider 인스턴스를 읽는다. 따라서 같은 ID를 두 번 읽었을 때 API가 불필요하게 두 번 호출되지 않는지도 확인할 수 있다.

```dart
test('같은 기기는 캐시를 재사용하고 다른 기기는 분리한다', () async {
  final fake = FakeDeviceRepository();
  final container = ProviderContainer(
    overrides: [deviceRepositoryProvider.overrideWithValue(fake)],
  );
  addTearDown(container.dispose);

  await container.read(deviceStateProvider('boiler-a').future);
  await container.read(deviceStateProvider('boiler-a').future);
  await container.read(deviceStateProvider('boiler-b').future);

  expect(fake.calls, ['boiler-a', 'boiler-b']);
});
```

다만 MQTT처럼 계속 들어오는 실시간 상태에는 `FutureProvider` 캐시만으로 부족하다. `StreamProvider.autoDispose.family`를 쓰더라도 테스트마다 새 `ProviderContainer`를 만들고, StreamController와 subscription을 반드시 닫아야 한다. `autoDispose`가 테스트 객체의 모든 리소스까지 대신 정리해준다고 생각하면 안 된다.

## 짧게 정리하면

Riverpod `family` 테스트의 기준은 “값이 나왔는가”가 아니라 “올바른 기기 ID의 경계 안에서 나왔는가”다.

- Repository Fake에 호출된 `deviceId`를 기록한다.
- 같은 컨테이너에서 최소 두 기기를 읽어 상태 오염을 확인한다.
- 같은 ID의 캐시 재사용과 다른 ID의 분리를 각각 검증한다.
- 테스트가 끝나면 `ProviderContainer`, StreamController, subscription을 정리한다.

기기 수가 늘어날수록 전역 Map을 직접 검사하는 테스트보다 family Provider를 실제 화면과 같은 방식으로 읽는 테스트가 오래 버틴다.
