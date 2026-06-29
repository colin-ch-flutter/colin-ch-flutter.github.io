---
layout: post
title: "Flutter shared_preferences 단위 테스트 — IoT 설정 저장소 격리 전략"
description: "shared_preferences를 단위 테스트에서 격리하는 두 가지 방법을 비교한다. setMockInitialValues로 빠르게 쓰는 방법과 FakePrefsStore 추상화로 DI에 깔끔하게 녹이는 방법을 IoT 앱 실전 예제로 정리했다."
date: 2026-06-29
tags: [Flutter, Dart, GetX, IoT, 스마트홈]
comments: true
share: true
---

![Flutter 앱 설정 저장소 테스트](https://images.unsplash.com/photo-1555949963-aa79dcee981c?w=800&q=80)

IoT 앱에서 `shared_preferences`는 BLE로 마지막에 연결한 기기 ID, MQTT 브로커 주소, 사용자 알림 설정처럼 가볍게 영속화할 값들을 담는다. 문제는 이게 플러그인 위에 얹혀 있어서, 그냥 unit test를 돌리면 `MissingPluginException`이 바로 튀어나온다는 것이다.

해결책은 두 가지다. 간단한 테스트라면 `SharedPreferences.setMockInitialValues()`로 30초 만에 끝낼 수 있고, 레이어를 제대로 격리하고 싶다면 추상 인터페이스를 뽑아서 `FakePrefsStore`를 DI에 주입한다. 어느 쪽이 적합한지 판단 기준부터 정리한다.

## MissingPluginException이 뜨는 이유

`shared_preferences`는 내부적으로 네이티브 채널을 통해 iOS의 `NSUserDefaults`, Android의 `SharedPreferences`에 접근한다. 플러터 테스트 환경은 네이티브 바이너리 없이 돌아가기 때문에 플러그인을 초기화하지 못하고 예외를 던진다.

```
MissingPluginException(No implementation found for method getAll 
on channel plugins.flutter.io/shared_preferences)
```

이 에러 자체는 명확한데, 이걸 보고 "그냥 integration test로 돌려야 하나?" 하고 결정을 못 내리는 경우가 있다. Unit test 레벨에서 충분히 격리 가능하다.

## 방법 1: setMockInitialValues — 간단한 케이스

`shared_preferences` 패키지 자체가 테스트용 mock 지원을 내장하고 있다. `setUp`에서 `SharedPreferences.setMockInitialValues()`를 호출하면 된다.

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  group('DeviceSettingsService', () {
    setUp(() {
      SharedPreferences.setMockInitialValues({
        'last_device_id': 'ble-boiler-001',
        'broker_host': 'xxxxxxx.iot.ap-northeast-2.amazonaws.com',
        'notification_enabled': true,
      });
    });

    test('마지막 연결 기기 ID를 읽어온다', () async {
      final prefs = await SharedPreferences.getInstance();
      expect(prefs.getString('last_device_id'), 'ble-boiler-001');
    });

    test('알림 설정을 저장하고 읽는다', () async {
      final prefs = await SharedPreferences.getInstance();
      await prefs.setBool('notification_enabled', false);
      expect(prefs.getBool('notification_enabled'), false);
    });
  });
}
```

`setMockInitialValues`에 넘기는 맵이 초기 상태다. `getInstance()` 호출 이후에 `setBool`, `setString` 등을 하면 인메모리에 반영되고, 다음 `getInstance()` 호출에서도 그 값이 유지된다. 각 테스트 사이에 `setUp`이 다시 실행되므로 상태 오염은 없다.

이 방법의 한계는 명확하다. `SharedPreferences`에 직접 의존하는 코드를 테스트하는 것이라 **Repository나 Service 레이어를 제대로 격리하지 못한다**. `SharedPreferences.getInstance()`가 코드 여기저기에 박혀 있으면 테스트 비용이 계속 올라간다.

## 방법 2: FakePrefsStore — DI 기반 격리

처음엔 "간단하게 쓰면 되지"라고 생각했는데, Controller에서 직접 `SharedPreferences.getInstance()`를 호출하는 코드가 쌓이기 시작하면 테스트 짜기가 점점 귀찮아진다. 결국 추상화를 뽑게 된다.

먼저 인터페이스를 정의한다.

```dart
abstract class IPrefsStore {
  Future<String?> getString(String key);
  Future<void> setString(String key, String value);
  Future<bool?> getBool(String key);
  Future<void> setBool(String key, bool value);
  Future<void> remove(String key);
}
```

실제 구현은 shared_preferences를 감싼다.

```dart
class SharedPrefsStore implements IPrefsStore {
  @override
  Future<String?> getString(String key) async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getString(key);
  }

  @override
  Future<void> setString(String key, String value) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(key, value);
  }

  @override
  Future<bool?> getBool(String key) async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getBool(key);
  }

  @override
  Future<void> setBool(String key, bool value) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool(key, value);
  }

  @override
  Future<void> remove(String key) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove(key);
  }
}
```

테스트용 Fake는 `Map`으로 만들면 된다.

```dart
class FakePrefsStore implements IPrefsStore {
  final Map<String, dynamic> _store = {};

  @override
  Future<String?> getString(String key) async => _store[key] as String?;

  @override
  Future<void> setString(String key, String value) async {
    _store[key] = value;
  }

  @override
  Future<bool?> getBool(String key) async => _store[key] as bool?;

  @override
  Future<void> setBool(String key, bool value) async {
    _store[key] = value;
  }

  @override
  Future<void> remove(String key) async => _store.remove(key);

  // 테스트에서 초기 상태 주입용
  void seed(Map<String, dynamic> values) => _store.addAll(values);
}
```

이제 `DeviceSettingsRepository`에 `IPrefsStore`를 주입하고, 테스트에서는 `FakePrefsStore`를 넣는다.

![FakePrefsStore DI 구조 다이어그램](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

```dart
class DeviceSettingsRepository {
  DeviceSettingsRepository(this._prefs);
  final IPrefsStore _prefs;

  static const _keyLastDeviceId = 'last_device_id';
  static const _keyBrokerHost = 'broker_host';
  static const _keyNotification = 'notification_enabled';

  Future<String?> getLastDeviceId() => _prefs.getString(_keyLastDeviceId);

  Future<void> saveLastDeviceId(String id) =>
      _prefs.setString(_keyLastDeviceId, id);

  Future<String?> getBrokerHost() => _prefs.getString(_keyBrokerHost);

  Future<bool> isNotificationEnabled() async =>
      (await _prefs.getBool(_keyNotification)) ?? true;

  Future<void> setNotificationEnabled(bool value) =>
      _prefs.setBool(_keyNotification, value);
}
```

테스트는 깔끔해진다.

```dart
void main() {
  late FakePrefsStore fakePrefs;
  late DeviceSettingsRepository repo;

  setUp(() {
    fakePrefs = FakePrefsStore();
    repo = DeviceSettingsRepository(fakePrefs);
  });

  group('DeviceSettingsRepository', () {
    test('기기 ID를 저장하고 읽는다', () async {
      await repo.saveLastDeviceId('ble-boiler-002');
      final id = await repo.getLastDeviceId();
      expect(id, 'ble-boiler-002');
    });

    test('저장된 기기 ID가 없으면 null을 반환한다', () async {
      final id = await repo.getLastDeviceId();
      expect(id, isNull);
    });

    test('알림 설정 기본값은 true다', () async {
      final enabled = await repo.isNotificationEnabled();
      expect(enabled, true);
    });

    test('알림을 끄면 false로 저장된다', () async {
      await repo.setNotificationEnabled(false);
      expect(await repo.isNotificationEnabled(), false);
    });

    test('초기 상태를 seed로 주입할 수 있다', () async {
      fakePrefs.seed({'last_device_id': 'ble-kitchen-001'});
      final id = await repo.getLastDeviceId();
      expect(id, 'ble-kitchen-001');
    });
  });
}
```

`seed()` 메서드가 있으면 복잡한 초기 상태를 한 줄에 세팅할 수 있어서 유용하다.

## GetX에서 DI 연결

실 코드에서는 `Get.put`으로 묶어주면 된다.

```dart
// main.dart 또는 bindings
void setupDependencies() {
  Get.put<IPrefsStore>(SharedPrefsStore());
  Get.put(DeviceSettingsRepository(Get.find<IPrefsStore>()));
}
```

테스트 환경에서는 `Get.put`으로 Fake를 주입하거나, Repository를 직접 생성자로 넘기면 GetX 없이도 테스트할 수 있다.

## 어느 방법을 쓸까

| 상황 | 추천 방법 |
|------|-----------|
| `SharedPreferences`를 직접 다루는 간단한 유틸 테스트 | `setMockInitialValues` |
| Repository/Service 레이어가 있고 DI로 주입되는 구조 | `FakePrefsStore` |
| Controller 단위 테스트에서 prefs 의존을 끊고 싶을 때 | `FakePrefsStore` |

스마트홈 앱처럼 설정값 종류가 많아지면 `setMockInitialValues`로 테스트마다 맵을 관리하는 게 귀찮아진다. Repository 레이어를 뽑아서 `IPrefsStore`를 DI하는 구조가 결국 유지보수에 낫다.

---

다음 편은 테스트 시리즈 전체를 돌아보는 **Flutter Unit Test 실전 시리즈 회고 — 13가지 격리 전략을 적용하고 얻은 것들**로 마무리할 예정이다.

**참고 링크**
- [shared_preferences 공식 문서](https://pub.dev/packages/shared_preferences)
- [Flutter 테스트 — 플러그인 격리](https://docs.flutter.dev/testing/plugins-in-tests)
