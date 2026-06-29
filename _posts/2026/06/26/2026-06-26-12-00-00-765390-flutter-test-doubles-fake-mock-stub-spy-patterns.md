---
layout: post
title: "Flutter 테스트 더블 완전 정복 — Fake·Stub·Mock·Spy, IoT 앱에서 언제 뭘 쓸까"
description: "테스트 더블 5종류(Dummy·Stub·Fake·Spy·Mock)의 차이를 Flutter IoT 앱 코드로 실제 비교했다. BLE·MQTT·Repository 레이어에서 각각 어떤 패턴이 적합한지 판단 기준을 정리한다."
date: 2026-06-26
tags: [Flutter, Dart, CleanArchitecture, BLE, MQTT, flutter_blue_plus, mqtt5_client]
comments: true
share: true
---

![소프트웨어 테스트 전략과 코드 품질](https://images.unsplash.com/photo-1555099962-4199c345e5dd?w=800&q=80)

이 시리즈에서 11편 동안 `FakeBleService`, `FakeMqttService`, `FakeDeviceRepository`를 줄곧 써왔다. 근데 정작 "왜 Mock 대신 Fake를 쓰냐"고 물으면 제대로 설명하기 어려웠다. 팀에 새 멤버가 들어왔을 때 "이건 Stub이야, Mock이야?" 소리 들으면 말문이 막힌다. 이번엔 테스트 더블 5종류를 이 시리즈 코드에 직접 대입해서 정리한다.

## 테스트 더블이 뭔가

테스트 더블(Test Double)은 촬영 현장의 스턴트맨 개념이다. 실제 배우(실제 객체) 대신 씬에 투입되는 대역. 종류가 다섯 개 있고, 각각 역할이 다르다.

| 종류 | 핵심 특징 | 언제 |
|------|-----------|------|
| Dummy | 존재만 함, 실제 사용 안 됨 | 매개변수 자리 채울 때 |
| Stub | 미리 정해진 값 반환 | 특정 상황 재현 |
| Fake | 단순 구현체가 실제로 동작 | 복잡한 의존성 대체 |
| Spy | 실제 동작하면서 호출 기록 | 호출 여부 검증 |
| Mock | 기대 동작 사전 등록 후 검증 | 상호작용 중심 검증 |

이렇게 표로 보면 다 아는 것 같은데, 실제 코드에서 헷갈린다.

## Dummy — 그냥 자리 채우기

```dart
// DeviceController 생성에 Logger가 필요하지만 이 테스트에서 로그는 관심 없음
class DummyLogger implements AppLogger {
  @override
  void log(String message) {} // 아무것도 안 함

  @override
  void error(String message, [Object? error]) {}
}

test('DeviceController 초기 상태는 disconnected', () {
  final controller = DeviceController(
    bleService: FakeBleService(),
    logger: DummyLogger(), // Logger 동작은 이 테스트에서 관심 없음
  );

  expect(controller.status.value, ConnectionStatus.disconnected);
});
```

DummyLogger는 실제로 호출되든 말든 상관없다. 그냥 컴파일 오류 안 나게 자리만 채우는 거다.

## Stub — 원하는 상황 강제 재현

```dart
class StubBleService implements BleService {
  final bool shouldConnect;
  StubBleService({required this.shouldConnect});

  @override
  Future<bool> connect(String deviceId) async => shouldConnect;

  // 나머지는 구현 안 해도 됨 (UnimplementedError 던짐)
  @override
  Stream<BleDevice> scan() => throw UnimplementedError();
}

test('연결 실패 시 에러 상태로 전환', () async {
  final controller = DeviceController(
    bleService: StubBleService(shouldConnect: false),
    logger: DummyLogger(),
  );

  await controller.connectDevice('device-001');

  expect(controller.status.value, ConnectionStatus.error);
});
```

Stub의 핵심은 **특정 시나리오를 강제**한다는 거다. 연결 실패 케이스를 테스트하고 싶을 때, 실제 BLE 기기 없이 `shouldConnect: false`로 박아버린다.

## Fake — 실제처럼 동작하는 단순 구현체

이 시리즈 내내 쓴 패턴이다. [BLE Wrapper 편]({% post_url 2026-06-24-10-00-00-654285-flutter-blue-plus-ble-unit-test-wrapper-isolation %})에서 만든 FakeBleService가 전형적인 예다.

```dart
class FakeBleService implements BleService {
  final _devices = <String, BleDevice>{};
  final _connectionState = <String, bool>{};

  // 실제 BLE처럼 동작하지만 실제 하드웨어 없이
  @override
  Stream<BleDevice> scan() async* {
    for (final device in _devices.values) {
      await Future.delayed(const Duration(milliseconds: 50));
      yield device;
    }
  }

  @override
  Future<bool> connect(String deviceId) async {
    if (!_devices.containsKey(deviceId)) return false;
    _connectionState[deviceId] = true;
    return true;
  }

  // 테스트에서 상태 조작용
  void addDevice(BleDevice device) => _devices[device.id] = device;
  bool isConnected(String id) => _connectionState[id] ?? false;
}
```

Stub과 달리 Fake는 내부 상태가 있다. `addDevice()` 하고 `connect()` 하면 실제로 연결 상태가 바뀐다. 여러 테스트 케이스를 하나의 Fake로 커버할 수 있어서 BLE·MQTT 같은 복잡한 의존성에 어울린다.

## Spy — 호출 기록하는 진짜 구현체

![테스트 스파이 패턴 — 실제 동작하면서 호출 기록](https://images.unsplash.com/photo-1526374965328-7f61d4dc18c5?w=800&q=80)

```dart
class SpyMqttService implements MqttService {
  final _realService = InMemoryMqttService(); // 실제 동작
  final publishedMessages = <String>[]; // 호출 기록
  int connectCallCount = 0;

  @override
  Future<void> connect() async {
    connectCallCount++;
    return _realService.connect();
  }

  @override
  Future<void> publish(String topic, String payload) async {
    publishedMessages.add('$topic:$payload');
    return _realService.publish(topic, payload);
  }

  @override
  Stream<MqttMessage> subscribe(String topic) => _realService.subscribe(topic);
}

test('보일러 온도 설정 시 MQTT publish 한 번만 호출', () async {
  final spy = SpyMqttService();
  final controller = BoilerController(mqttService: spy);

  await controller.setTemperature(24);

  expect(spy.publishedMessages.length, 1);
  expect(spy.publishedMessages.first, contains('boiler/set'));
});
```

Spy는 실제 동작을 유지하면서 "몇 번 호출됐나", "어떤 인자로 호출됐나"를 기록한다. 특히 MQTT publish가 중복 호출되는 버그를 잡을 때 유용하다. 처음엔 이 패턴을 몰라서 로그에 print() 박아서 디버깅했는데, 그냥 Spy 쓰면 됐던 거다.

## Mock — 기대 동작 등록하고 검증

mocktail을 쓰면 이렇게 된다.

```dart
import 'package:mocktail/mocktail.dart';

class MockMqttService extends Mock implements MqttService {}

test('연결 실패 시 재시도 로직 동작', () async {
  final mock = MockMqttService();

  // 기대 동작 등록
  when(() => mock.connect())
      .thenThrow(MqttConnectionException('timeout'));

  final controller = DeviceController(mqttService: mock);
  await controller.initialize();

  // 상호작용 검증
  verify(() => mock.connect()).called(1);
  expect(controller.hasError, true);
});
```

Mock의 핵심은 **상호작용(interaction) 검증**이다. "이 메서드가 정확히 이 방식으로 호출됐는가"에 집중한다. Fake는 내부 상태 변화를 검증하고, Mock은 호출 패턴을 검증한다.

문서에는 잘 안 나와 있는데, mocktail에서 `when()` 등록 안 하고 호출하면 `MissingStubError`가 터진다. 처음엔 이게 뭔가 싶었다.

## IoT 앱에서 어떤 패턴을 쓸까

솔직히 정리해보면:

**BLE (flutter_blue_plus)**
→ Fake를 쓴다. scan·connect·disconnect가 연결되어 있어서 상태 기반 동작이 필요하다. Stub으론 복잡한 시나리오 커버가 안 된다.

**MQTT (mqtt5_client)**
→ Fake가 기본, publish 중복 검증이 필요하면 Spy 추가.

**Repository 레이어**
→ Fake Repository. 실제 DB(Realm, SharedPreferences) 없이 메모리로 CRUD 다 동작.

**단순 유틸리티 서비스 (Logger, Analytics)**
→ Dummy로 충분. 동작 여부가 테스트 관심사가 아님.

**에러 경로 테스트**
→ Stub 또는 Mock. "이 조건일 때 예외를 던진다"는 시나리오는 Stub이 직관적이다.

```dart
// 경험상 선택 기준
if (복잡한_상태_변화가_필요) {
  return Fake();
} else if (호출_횟수_검증이_목적) {
  return Spy();
} else if (특정_예외_시나리오_재현) {
  return Stub(); // 또는 Mock
} else if (자리만_채우면_됨) {
  return Dummy();
}
```

## mocktail vs mockito

처음엔 mockito를 썼다. `@GenerateMocks([BleService])` 어노테이션 붙이고 `flutter pub run build_runner build` 돌리는 방식. 근데 코드 변경할 때마다 다시 생성해야 해서 번거롭다.

mocktail은 코드 생성 없이 `class MockBleService extends Mock implements BleService {}` 한 줄로 끝난다. 이 시리즈에서 Fake를 주로 쓴 건 mocktail도 아니고 mockito도 아닌 이유가 있다.

BLE랑 MQTT는 Mock으로 만들면 `when()` 등록해야 할 메서드가 너무 많다. scan은 Stream 반환이라 타입 매칭도 복잡하다. Fake 하나 만들어두면 여러 테스트에서 재사용 가능하고 setup 코드도 훨씬 짧다.

```dart
// Mock 방식 — BLE scan 테스트 setup이 복잡
when(() => mockBleService.scan()).thenAnswer(
  (_) => Stream.fromIterable([device1, device2]),
);

// Fake 방식 — 훨씬 직관적
fakeBleService.addDevice(device1);
fakeBleService.addDevice(device2);
```

두 줄짜리 setup이 이기는 거다.

---

다음 편은 이 시리즈를 마무리하는 회고다. 12편 동안 뭘 만들었고, 테스트 커버리지가 실제로 코드 품질에 어떤 영향을 줬는지 돌아본다.

**참고 링크**
- [mocktail 공식 문서](https://pub.dev/packages/mocktail)
- [mockito 공식 문서](https://pub.dev/packages/mockito)
- [Flutter 공식 테스팅 가이드](https://docs.flutter.dev/testing/overview)
- [BLE Wrapper 패턴 편]({% post_url 2026-06-24-10-00-00-654285-flutter-blue-plus-ble-unit-test-wrapper-isolation %})
- [MQTT Fake 서비스 편]({% post_url 2026-06-24-14-00-00-666630-mqtt5-client-unit-test-fake-service-isolation %})
