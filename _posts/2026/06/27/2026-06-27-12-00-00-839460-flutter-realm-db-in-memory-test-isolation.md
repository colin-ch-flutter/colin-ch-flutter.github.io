---
layout: post
title: "Flutter Realm DB 테스트 격리 — 인메모리 Realm으로 Repository 단위 테스트"
description: "realm.write() 내부 메서드는 Mockito로 mock이 안 된다. Configuration.inMemory()로 진짜 Realm을 인메모리로 열어 DeviceRepository를 실제처럼 단위 테스트하는 방법을 정리했다."
date: 2026-06-27
tags: [Flutter, Dart, realm, CleanArchitecture, Repository패턴]
comments: true
share: true
---

# Flutter Realm DB 테스트 격리 — 인메모리 Realm으로 Repository 단위 테스트

![데이터베이스 테스트 격리 전략](https://images.unsplash.com/photo-1544383835-bda2bc66a55d?w=800&q=80)

IoT 앱에서 기기 상태를 로컬 캐시로 저장할 때 Realm을 쓴다. 빠르고 Flutter 친화적이고, MQTT로 받은 실시간 데이터를 오프라인에서도 보여줄 수 있어서다. 근데 이걸 테스트하려고 하면 벽에 부딪힌다. `realm.write()` 블록 안에서 호출되는 메서드는 Mockito로 mock이 안 된다.

결론부터 말하면: **Realm은 mock하지 말고 `Configuration.inMemory()`로 진짜 Realm을 인메모리로 열어라.** 이렇게 하면 파일 I/O 없이 실제 Realm 동작을 테스트에서 그대로 쓸 수 있다.

## 왜 Realm은 mock이 안 되나

처음엔 당연히 이렇게 하려 했다.

```dart
// 이렇게 하면 안 된다
class MockRealm extends Mock implements Realm {}

test('기기 상태 저장', () {
  final mockRealm = MockRealm();
  when(() => mockRealm.write(any())).thenReturn(null); // ❌ write()는 generic callback
  // ...
});
```

`realm.write<T>(T Function() fn)` 는 내부적으로 `fn`을 실행하고 그 결과를 반환한다. Mock으로 `write()`를 가로채도 실제 콜백(`realm.add(device)`)은 실행되지 않는다. 타입도 안 맞고, 반환값도 못 꺼낸다.

[realm GitHub 이슈 #870](https://github.com/realm/realm-dart/issues/870)에서도 같은 질문이 올라왔고, 공식 답변은 "인메모리 Realm을 쓰라"였다.

## 인메모리 Realm 설정

`Configuration.inMemory()` 를 쓰면 실제 파일을 만들지 않고 Realm을 열 수 있다. 각 테스트마다 고유한 path를 줘야 테스트 간 데이터가 섞이지 않는다.

```dart
import 'package:realm/realm.dart';
import 'package:test/test.dart';

Realm openTestRealm() {
  final config = Configuration.inMemory(
    [DeviceState.schema, SpaceState.schema],
    path: 'test_${DateTime.now().microsecondsSinceEpoch}.realm',
  );
  return Realm(config);
}
```

path에 타임스탬프나 UUID를 붙이는 이유가 있다. `Configuration.inMemory()`는 같은 path면 같은 인스턴스를 공유한다. 병렬 테스트나 setUp/tearDown 타이밍에 따라 이전 테스트 데이터가 남아있는 경우가 생긴다. 처음에 이걸 몰라서 테스트가 순서 의존적으로 통과하고 혼자 CI에서만 실패했다.

## DeviceRepository 테스트 구조

IoT 앱의 `DeviceRepository`는 MQTT 메시지를 받아 기기 상태를 Realm에 저장하고, UI가 Realm 결과를 구독하는 구조다.

```dart
// lib/data/repository/device_repository_impl.dart
class DeviceRepositoryImpl implements DeviceRepository {
  final Realm _realm;

  DeviceRepositoryImpl(this._realm);

  @override
  void saveDeviceState(String deviceId, bool isOn, double temperature) {
    final existing = _realm.find<DeviceState>(deviceId);
    _realm.write(() {
      if (existing != null) {
        existing.isOn = isOn;
        existing.temperature = temperature;
        existing.updatedAt = DateTime.now();
      } else {
        _realm.add(DeviceState(deviceId, isOn, temperature, DateTime.now()));
      }
    });
  }

  @override
  DeviceState? getDeviceState(String deviceId) {
    return _realm.find<DeviceState>(deviceId);
  }

  @override
  Stream<RealmResultsChanges<DeviceState>> watchDevices() {
    return _realm.all<DeviceState>().changes;
  }
}
```

테스트는 이렇게 쓴다.

```dart
// test/data/repository/device_repository_impl_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:realm/realm.dart';
import 'package:your_app/data/repository/device_repository_impl.dart';
import 'package:your_app/data/model/device_state.dart';

void main() {
  late Realm realm;
  late DeviceRepositoryImpl repository;

  setUp(() {
    realm = openTestRealm();
    repository = DeviceRepositoryImpl(realm);
  });

  tearDown(() {
    realm.close(); // 닫아야 path 재사용이 안 생긴다
  });

  group('DeviceRepository 저장', () {
    test('새 기기 상태를 저장한다', () {
      repository.saveDeviceState('boiler-01', true, 72.5);

      final result = repository.getDeviceState('boiler-01');
      expect(result, isNotNull);
      expect(result!.isOn, isTrue);
      expect(result.temperature, equals(72.5));
    });

    test('같은 deviceId면 상태를 업데이트한다', () {
      repository.saveDeviceState('boiler-01', true, 72.5);
      repository.saveDeviceState('boiler-01', false, 20.0);

      final result = repository.getDeviceState('boiler-01');
      expect(result!.isOn, isFalse);
      expect(result.temperature, equals(20.0));

      // upsert이므로 레코드가 1개여야 한다
      expect(realm.all<DeviceState>().length, equals(1));
    });

    test('다른 deviceId는 별도 레코드로 저장된다', () {
      repository.saveDeviceState('boiler-01', true, 72.5);
      repository.saveDeviceState('aircon-01', false, 26.0);

      expect(realm.all<DeviceState>().length, equals(2));
    });
  });
}
```

![Realm 인메모리 테스트 실행 흐름](https://images.unsplash.com/photo-1629654297299-c8506221ca97?w=800&q=80)

## Stream 변경 감지 테스트

Realm의 `results.changes` Stream을 구독하는 경우도 테스트할 수 있다.

```dart
test('기기 상태 변경 시 Stream 이벤트가 발생한다', () async {
  final stream = repository.watchDevices();
  
  // 첫 번째 이벤트는 초기 구독 (빈 결과)
  final subscription = stream.listen(null);
  
  repository.saveDeviceState('boiler-01', true, 72.5);
  
  final changes = await stream.first;
  expect(changes.inserted.length, equals(1));
  
  await subscription.cancel();
});
```

근데 여기서 함정이 있다. Realm의 변경 알림은 Dart 이벤트 루프를 한 사이클 돌아야 전달된다. `fakeAsync` 안에서 테스트하면 `pump()`를 한 번 써줘야 알림이 온다. 이전 포스트에서 다룬 [비동기 테스트 패턴]({% post_url 2026-06-26-14-00-00-777735-flutter-fake-async-stream-timer-unit-test %})과 같은 맥락이다.

## tearDown에서 realm.close()를 빠뜨리면 생기는 일

초반에 `tearDown`에서 `realm.close()`를 안 써줬더니 두 번째 테스트부터 이런 오류가 났다.

```
RealmError: The Realm at path '...' already has an open Realm without sync.
```

인메모리 Realm도 "열려있는 인스턴스"를 추적한다. 같은 path로 두 번 열면 에러가 난다. `setUp`에서 타임스탬프 path를 써도, 이미 닫히지 않은 인스턴스가 살아있으면 문제가 생긴다. 반드시 `tearDown(() => realm.close())` 를 넣어야 한다.

## Schema 등록 주의사항

인메모리 Realm을 열 때 `Configuration.inMemory(schemas, ...)` 에 테스트에서 쓰는 **모든** RealmObject 스키마를 넣어야 한다.

```dart
// DeviceState가 SpaceState를 참조하면 SpaceState도 포함해야 한다
final config = Configuration.inMemory(
  [DeviceState.schema, SpaceState.schema], // 연관 스키마 전부
  path: 'test_${DateTime.now().microsecondsSinceEpoch}.realm',
);
```

`DeviceState`가 `SpaceState`를 참조하는데 스키마를 빠뜨리면 실행 시점에 "Missing class" 계열 오류가 나서 디버깅하기 번거롭다.

## BLE·MQTT 격리와 조합하기

실제 IoT 앱 테스트에서는 MQTT 메시지 → Repository 저장 → UI 반영 흐름 전체를 검증한다.

```dart
test('MQTT 메시지 수신 시 기기 상태가 Realm에 저장된다', () async {
  final fakeMqtt = FakeMqttService(); // 이전 시리즈에서 만든 Fake
  final controller = DeviceController(
    mqttService: fakeMqtt,
    deviceRepository: repository, // 인메모리 Realm Repository
  );

  await controller.initialize();
  fakeMqtt.simulateMessage(
    topic: 'home/boiler-01/status',
    payload: '{"isOn": true, "temperature": 72.5}',
  );

  await Future.microtask(() {}); // 이벤트 루프 한 사이클

  final saved = repository.getDeviceState('boiler-01');
  expect(saved?.isOn, isTrue);
});
```

이렇게 하면 네트워크도, 실기기도, 파일 I/O도 없이 전체 데이터 흐름을 단위 테스트로 검증할 수 있다.

---

다음 편은 `SecureStorage` 격리 테스트다. JWT 토큰을 `flutter_secure_storage`에 저장하는 코드는 Platform Channel을 타기 때문에 인메모리 대안이나 Fake 구현이 필요하다. Realm 격리와 비슷한 접근이지만, 스토리지 특성상 다루는 포인트가 다르다.

**참고 링크**
- [realm Flutter 패키지](https://pub.dev/packages/realm)
- [realm-dart GitHub 이슈 #870 — unit tests using realm](https://github.com/realm/realm-dart/issues/870)
- [How to test Realm in Flutter — Medium](https://medium.com/@midorisaku/how-to-test-realm-in-flutter-c82bff83f32c)
