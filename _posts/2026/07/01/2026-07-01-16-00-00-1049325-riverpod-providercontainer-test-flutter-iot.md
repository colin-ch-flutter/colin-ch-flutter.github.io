---
layout: post
title: "Flutter IoT Riverpod 테스트 - ProviderContainer로 상태 전이 검증하기"
description: "Flutter IoT 앱에서 Riverpod ProviderContainer로 Flutter BLE와 MQTT 상태 전이를 테스트하고 overrideWith로 Fake 의존성을 주입하는 방법을 정리했다."
date: 2026-07-01
tags: [Flutter, Dart, BLE, MQTT, IoT]
comments: true
share: true
---
![Flutter IoT Riverpod 테스트](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

Flutter IoT 앱에서 Riverpod으로 넘어오면 테스트 방식도 바뀐다. Flutter 스마트홈 앱의 Flutter BLE 스캔 상태나 MQTT 연결 상태를 GetX Controller처럼 직접 생성해서 검증하는 대신, `ProviderContainer`를 만들고 Provider 의존성을 갈아끼운 뒤 상태 전이를 읽는 쪽이 더 깔끔했다. 처음엔 `ref.watch`가 화면 안에서만 의미 있는 줄 알았는데 아니었다.

[GetX 상태관리]({% post_url 2026-05-02-09-21-39-037035-getx-state-management-routing-flutter-iot %})를 쓸 때는 Controller를 만들고 Fake Repository를 넣으면 끝이었다. 문제는 Riverpod으로 옮기면서 상태 소유자가 Controller 인스턴스가 아니라 Provider 그래프가 됐다는 점이다. 이걸 무시하고 Notifier만 직접 생성하니 `ref.read()`가 필요한 순간부터 테스트가 깨졌다.

## ProviderContainer를 테스트 루트로 둔다

Riverpod 테스트에서는 `ProviderContainer`가 작은 앱 루트 역할을 한다. 실제 앱의 `ProviderScope` 대신 테스트용 컨테이너를 만들고, BLE나 MQTT Repository는 `overrideWithValue`로 바꿔 넣는다.

MQTT 연결 성공 상태를 검증하는 코드는 이런 모양이 됐다.

```dart
test('MQTT 연결 성공이면 connected 상태가 된다', () async {
  final fakeMqtt = FakeMqttService(connectResult: true);
  final container = ProviderContainer(
    overrides: [
      mqttServiceProvider.overrideWithValue(fakeMqtt),
    ],
  );
  addTearDown(container.dispose);

  await container.read(mqttConnectionProvider.notifier).connect();

  final state = container.read(mqttConnectionProvider);
  expect(state.isConnected, true);
  expect(state.errorMessage, isNull);
});
```

핵심은 Notifier를 직접 new 하지 않는 것이다. Notifier 내부에서 다른 Provider를 읽는 순간 직접 생성 방식은 실제 구조와 달라진다. 테스트가 통과해도 앱에서는 실패할 수 있다.

## BLE 테스트는 Stream을 Fake로 고정한다

Flutter BLE 테스트도 같은 방식으로 풀었다. `flutter_blue_plus 사용법` 예제처럼 실제 스캔을 돌리면 unit test가 아니다. iOS 실기기에서만 의미 있는 동작도 있고, CI에서는 Bluetooth 권한 자체가 없다. 그래서 BLE Wrapper Provider를 Fake로 바꾸고 Stream을 내가 원하는 순서로 흘려보냈다.

```dart
final controller = StreamController<List<BleDevice>>();
final fakeBle = FakeBleService(scanStream: controller.stream);

final container = ProviderContainer(
  overrides: [
    bleServiceProvider.overrideWithValue(fakeBle),
  ],
);

controller.add([BleDevice(id: 'boiler-01', rssi: -58)]);

final devices = await container.read(bleScanProvider.future);
expect(devices.first.id, 'boiler-01');
```

해보니 됐다. 다만 StreamProvider는 타이밍을 대충 두면 테스트가 흔들린다. `pumpAndSettle`로 화면을 기다리는 Widget Test와 다르게, unit test에서는 Future를 명확히 기다리거나 `fakeAsync`로 시간을 고정하는 편이 낫다.

## ProviderObserver는 로그 대신 검증 도구다

[테스트 커버리지 전략]({% post_url 2026-06-26-16-00-00-790080-flutter-test-coverage-strategy-threshold %})을 정리할 때 커버리지 숫자보다 실패 원인을 빨리 찾는 구조가 더 중요하다고 느꼈다. Riverpod에서는 `ProviderObserver`가 그 역할을 꽤 해준다. MQTT 연결 끊김, BLE 재연결, 로딩 상태 전환을 기록해두면 예상하지 못한 중간 상태를 잡기 쉽다.

솔직하게 정리해보면 ProviderContainer 테스트는 처음엔 GetX보다 번거롭다. 대신 앱 구조와 거의 같은 Provider 그래프 위에서 검증한다는 장점이 있다. IoT 앱처럼 BLE, MQTT, 인증, 캐시가 얽힌 화면에서는 이 차이가 크다. 상태를 직접 만지는 테스트보다 Provider를 통해 읽고 바꾸는 테스트가 오래 버텼다.

짧게 남기면 이렇다. Riverpod 테스트의 시작점은 Notifier 생성자가 아니라 `ProviderContainer`다. 외부 의존성은 override로 바꾸고, BLE와 MQTT 같은 비동기 상태는 Fake Stream과 명시적인 await로 고정한다. 그래야 Flutter IoT 테스트가 로컬과 CI에서 같은 이유로 통과하고 같은 이유로 실패한다.
