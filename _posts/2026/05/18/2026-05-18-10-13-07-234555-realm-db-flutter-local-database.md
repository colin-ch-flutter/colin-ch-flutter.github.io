---
layout: post
title: "Realm DB 도입기 - Flutter에서 로컬 DB 사용하기"
description: " "
date: 2026-05-18
tags: [Flutter, Realm, 로컬DB, 데이터베이스]
comments: true
share: true
---
![데이터베이스](https://images.unsplash.com/photo-1544383835-bda2bc66a55d?w=800&q=80)

## 왜 Realm을 선택했나

Flutter에서 로컬 데이터를 저장하는 방법은 여러 가지다:

| 방법 | 장점 | 단점 |
|------|------|------|
| SharedPreferences | 간단 | 키-값만 가능, 복잡한 데이터 못 씀 |
| SQLite (sqflite) | 익숙한 SQL | 스키마 정의가 번거로움 |
| Hive | 빠름, 타입 안전 | 관계형 데이터 처리 약함 |
| Realm | 객체 지향, 빠름, 관계 지원 | 파일 크기가 큼 |

IoT 기기 기기 정보, 사용자 설정, 스케줄 등 서로 관계가 있는 데이터가 많아서 Realm을 선택했다. Realm은 객체를 그대로 저장하고 조회할 수 있어서 Flutter 코드와 자연스럽게 맞는다.

## Realm 설정

```yaml
dependencies:
  realm: ^20.2.0
  
dev_dependencies:
  realm: ^20.2.0  # 코드 생성용
  build_runner: ^2.4.15
```

## 스키마 정의

`RealmModel`로 데이터 모델을 정의한다:

```dart
// data/models/db/iot_device_device_realm.dart
part 'iot_device_device_realm.realm.dart';

@RealmModel()
class _IoT DeviceDeviceRealm {
  @PrimaryKey()
  late String id;
  
  late String nickname;
  late String modelName;
  late bool isOnline;
  late double? lastTemperature;
  late String? lastMode;
  late DateTime? lastUpdatedAt;
  late String spaceId;
}
```

`build_runner`로 코드를 생성한다:

```bash
flutter pub run build_runner build
```

생성된 `iot_device_device_realm.realm.dart`에 실제 Realm 클래스가 만들어진다.

## CRUD 구현

```dart
// data/services/db_service.dart
class DbService {
  late Realm _realm;
  
  Future<void> init() async {
    final config = Configuration.local(
      [IoT DeviceDeviceRealm.schema, UserSettingsRealm.schema],
      schemaVersion: 1,
      migrationCallback: _onMigration,
    );
    _realm = Realm(config);
  }
  
  // 저장
  Future<void> saveDevice(IoT DeviceDevice device) async {
    final realmObj = IoT DeviceDeviceRealm(
      device.id,
      device.nickname,
      device.modelName,
      device.isOnline,
      spaceId: device.spaceId,
    );
    _realm.write(() => _realm.add(realmObj, update: true));
  }
  
  // 조회
  IoT DeviceDevice? getDevice(String deviceId) {
    final obj = _realm.find<IoT DeviceDeviceRealm>(deviceId);
    if (obj == null) return null;
    return IoT DeviceDeviceMapper.fromRealm(obj);
  }
  
  // 목록 조회
  List<IoT DeviceDevice> getDevicesBySpace(String spaceId) {
    return _realm
        .query<IoT DeviceDeviceRealm>('spaceId == \$0', [spaceId])
        .map(IoT DeviceDeviceMapper.fromRealm)
        .toList();
  }
  
  // 삭제
  Future<void> deleteDevice(String deviceId) async {
    final obj = _realm.find<IoT DeviceDeviceRealm>(deviceId);
    if (obj != null) {
      _realm.write(() => _realm.delete(obj));
    }
  }
}
```

## 마이그레이션 처리

스키마가 바뀌면 마이그레이션이 필요하다. `schemaVersion`을 올리고 콜백을 작성한다:

```dart
const config = Configuration.local(
  [...schemas],
  schemaVersion: 2,  // 버전 올리기
  migrationCallback: (migration, oldSchemaVersion) {
    if (oldSchemaVersion < 2) {
      // 버전 1 → 2 마이그레이션
      // 예: 새 컬럼 추가 (기본값 설정)
      migration.newRealm
          .query<IoT DeviceDeviceRealm>('TRUEPREDICATE')
          .forEach((obj) {
        obj.newField = 'default_value';
      });
    }
  },
);
```

## Realm 주의사항

Realm 객체는 특정 스레드에 묶여있다. 다른 스레드에서 접근하면 오류가 난다.

Flutter는 기본적으로 단일 스레드(Isolate)에서 동작하지만, `compute()`나 별도 Isolate를 사용할 때 주의해야 한다:

```dart
// 잘못된 예: compute 안에서 Realm 객체 사용
final result = await compute(heavyTask, realmObject);  // ❌

// 올바른 예: Realm 조회는 메인 Isolate에서
final data = getFromRealm();  // 메인 스레드에서 먼저 데이터 추출
final result = await compute(heavyTask, data);  // ✅
```

---

다음 편은 Repository 패턴으로 데이터 레이어를 깔끔하게 구성하는 방법이다.
