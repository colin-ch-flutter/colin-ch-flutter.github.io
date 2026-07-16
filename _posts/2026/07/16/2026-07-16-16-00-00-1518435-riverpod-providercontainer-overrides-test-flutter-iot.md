---
layout: post
title: "Flutter IoT Riverpod ProviderContainer override 테스트 - BLE·MQTT 상태 오염 막기"
description: "Flutter IoT 앱에서 Riverpod ProviderContainer override로 BLE와 MQTT Fake 구현을 테스트별로 격리하고, 상태 오염과 dispose 누수를 검증하는 방법을 정리했다."
date: 2026-07-16
tags: [Flutter, Dart, Riverpod, BLE, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod ProviderContainer override 테스트와 BLE MQTT 격리](https://images.unsplash.com/photo-1558002038-1055907df827?w=1200&q=80)

Flutter IoT 테스트에서 Riverpod `ProviderContainer`를 재사용하면 BLE 연결 상태와 MQTT 메시지가 다음 테스트로 새어 나갈 수 있다. 테스트마다 Fake Repository를 `override`한 새 컨테이너를 만들고, 끝에서 반드시 `dispose`하는 방식이 가장 단순하고 확실했다.

## 같은 Fake를 써도 테스트가 흔들린 이유

보일러 카드 테스트를 여러 개 만들면서 공통 `container`를 두고 시작했다. 첫 테스트에서 `connected`를 읽은 뒤 두 번째 테스트에서 `disconnected`를 기대했는데, 이전 Provider 상태가 남아 간헐적으로 실패했다. MQTT `StreamController`도 이전 구독을 계속 들고 있어 메시지가 두 번 처리됐다.

| 방식 | BLE·MQTT 상태 | 실패 원인 |
| --- | --- | --- |
| 테스트 전체에서 컨테이너 1개 | 공유됨 | 이전 Provider 캐시와 listener 잔존 |
| 테스트마다 새 컨테이너 | 격리됨 | 초기 상태를 명시해야 함 |
| 새 컨테이너 + `dispose` | 격리·정리됨 | 스트림 누수까지 검증 가능 |

핵심은 Fake 객체만 새로 만드는 게 아니었다. Provider가 보관하는 캐시와 listener의 생명주기까지 테스트 경계 안에 넣어야 했다.

## 테스트 전용 ProviderContainer를 만든다

아래 헬퍼는 실제 BLE·MQTT 구현체 대신 테스트용 Repository를 주입하고, 테스트가 끝나면 컨테이너를 정리한다.

```dart
ProviderContainer createTestContainer({
  required FakeDeviceRepository deviceRepository,
  required FakeMqttRepository mqttRepository,
}) {
  final container = ProviderContainer(
    overrides: [
      deviceRepositoryProvider.overrideWithValue(deviceRepository),
      mqttRepositoryProvider.overrideWithValue(mqttRepository),
    ],
  );

  addTearDown(() {
    container.dispose();
    mqttRepository.close();
  });

  return container;
}
```

`addTearDown`은 `test()` 또는 `testWidgets()`의 본문에서 호출해야 한다. 헬퍼 안에서 등록하므로 호출한 테스트에만 정리가 붙는다. `ProviderContainer.dispose()`가 Fake 내부에서 만든 `StreamController`까지 자동으로 닫아준다고 기대하면 안 된다. 그 리소스의 소유자가 Fake라면 `close()`도 별도로 호출해야 한다.

## override가 실제 Provider에 적용됐는지 확인한다

컨테이너를 만들었다고 끝내지 말고, 첫 읽기부터 Fake가 사용됐는지 확인한다. 실제 Repository에 연결되면 테스트가 느려질 뿐 아니라 개발 장비의 BLE 상태나 브로커 응답에 따라 결과가 달라진다.

```dart
test('기기 상태는 테스트 Fake에서 읽는다', () async {
  final fakeDevice = FakeDeviceRepository(
    initial: DeviceState.connected('boiler-1', temperature: 21.5),
  );
  final fakeMqtt = FakeMqttRepository();
  final container = createTestContainer(
    deviceRepository: fakeDevice,
    mqttRepository: fakeMqtt,
  );

  final state = await container.read(deviceStateProvider('boiler-1').future);

  expect(state.temperature, 21.5);
  expect(fakeDevice.readCount, 1);
});
```

여기서 중요한 건 테스트마다 `FakeDeviceRepository`를 새로 만든다는 점이다. `overrideWithValue`에 넣은 객체는 Provider가 대신 복제해주지 않는다. 같은 Fake를 여러 테스트가 공유하면 내부 `readCount`, `StreamController`, 명령 이력까지 함께 공유된다.

## 주의할 경계

`autoDispose` Provider는 listener가 사라지면 정리 대상이 되지만, 컨테이너 자체를 계속 살려두면 테스트가 기대한 시점에 dispose가 일어나지 않을 수 있다. 스트림 Provider를 읽은 테스트에서는 `container.listen`의 subscription도 닫고, Fake MQTT의 연결 종료 여부를 별도 카운터로 확인하는 게 좋다.

또 `ProviderScope`의 `overrides`와 `ProviderContainer`의 `overrides`를 섞어 쓰면 어느 레이어의 Fake가 적용됐는지 헷갈린다. 위젯 테스트는 `ProviderScope`, 순수 로직 테스트는 `ProviderContainer`로 경계를 정하고 한 테스트 안에서 같은 Provider를 두 방식으로 덮지 않는 편이 안전했다.

짧게 정리하면:

- 테스트마다 새 `ProviderContainer`를 만든다.
- BLE·MQTT Fake와 내부 스트림을 테스트별로 소유한다.
- `dispose()`와 Fake의 `close()`를 `addTearDown`에 등록한다.
- override가 적용됐는지 호출 횟수와 초기 상태로 확인한다.
