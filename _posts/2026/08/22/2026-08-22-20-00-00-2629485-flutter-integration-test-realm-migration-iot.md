---
layout: post
title: "Flutter integration_test Realm 마이그레이션 검증 - IoT 기기 상태 유실 막기"
description: "Flutter integration_test로 Realm 스키마 버전이 올라갈 때 IoT 기기와 스마트홈 설정이 유지되는지 검증하고, 마이그레이션 누락으로 로컬 상태가 사라지는 문제를 잡는 방법을 정리했다."
date: 2026-08-22
tags: [Flutter, Dart, Realm, integration_test, IoT, 스마트홈]
comments: true
share: true
---
![Flutter integration_test Realm 마이그레이션으로 IoT 로컬 상태 검증](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Flutter integration_test로 새 설치만 확인하면 Realm 마이그레이션 버그를 놓치기 쉽다. 스마트홈 앱은 이미 등록한 보일러, 별명, 자동화 설정을 로컬에 저장하고 있는데, 앱 업데이트 뒤 스키마가 바뀌면서 기기 카드가 전부 사라지는 문제가 있었다. 결론은 **빈 DB가 아니라 이전 버전 데이터로 앱을 시작하는 테스트가 하나는 있어야 한다**는 것이다.

## 새로 설치한 테스트는 마이그레이션을 검사하지 못한다

처음에는 integration_test가 실행될 때마다 Realm 파일을 지웠다. 하지만 실제 사용자는 앱을 업데이트한다. `Device`에 `roomId`를 추가한 뒤 기존 계정으로 실행하자, migration callback에서 필드를 채우지 않아 거실 보일러가 “알 수 없음” 목록으로 밀려났다.

| 시작 상태 | 확인할 것 | 실패하면 생기는 문제 |
|---|---|---|
| 빈 Realm | 새 스키마로 생성되는지 | 업데이트 회귀를 못 잡음 |
| v3 데이터 | 기존 필드가 보존되는지 | 기기·방 이름 유실 |
| v3 + MQTT 이벤트 | 마이그레이션 후 구독되는지 | 화면은 복원됐지만 상태가 멈춤 |

## 테스트용 이전 버전 Realm 파일을 주입한다

테스트 앱이 시작되기 전에 fixture를 복사해 두면 실제 업데이트와 비슷한 조건을 만들 수 있다. 아래 코드는 v3 fixture를 앱의 문서 디렉터리에 배치하고, 앱은 현재 스키마 버전으로 Realm을 연다.

```dart
Future<void> seedLegacyRealm() async {
  final directory = await getApplicationDocumentsDirectory();
  final target = File('${directory.path}/smart_home.realm');
  final fixture = File('test_fixtures/realm/v3/smart_home.realm');

  if (await target.exists()) {
    await target.delete();
  }
  await target.writeAsBytes(await fixture.readAsBytes());
}

testWidgets('v3 Realm 업데이트 후 기존 기기와 방 이름을 보존한다',
    (tester) async {
  await seedLegacyRealm();
  await launchTestApp(useFakeMqtt: true);

  await tester.pumpAndSettle();
  expect(find.text('거실 보일러'), findsOneWidget);
  expect(find.text('거실'), findsOneWidget);
  expect(find.text('마이그레이션 실패'), findsNothing);
});
```

fixture를 매 테스트 전에 다시 복사해야 한다. 한 번 마이그레이션된 파일을 재사용하면 두 번째 실행부터는 이미 최신 스키마라 테스트가 무의미해진다. `useFakeMqtt: true`는 브로커가 아니라 DB 변환과 화면 바인딩만 확인하기 위한 선택이다.

## migration callback에서 기본값을 명시한다

새 필드가 nullable이 아니라면 기존 객체에 어떤 값을 넣을지 정해야 한다. 아래처럼 `roomId`를 일괄적으로 빈 문자열로 넣으면 화면에서 미분류 기기로 표시할 수 있고, 마이그레이션 도중 예외도 피할 수 있다.

```dart
final config = Configuration.local(
  [Device.schema, Room.schema],
  schemaVersion: 4,
  migrationCallback: (migration, oldSchemaVersion) {
    if (oldSchemaVersion < 4) {
      migration.renameProperty<Device>('Device', 'name', 'displayName');
      migration.enumerate<Device>((oldObject, newObject) {
        newObject.roomId = oldObject.roomId ?? 'unassigned';
      });
    }
  },
);
```

Realm API 버전에 따라 migration 객체의 세부 타입은 달라질 수 있다. 중요한 건 callback보다 **이전 데이터의 각 필드가 어떤 값으로 변환되는지 기대값으로 고정하는 것**이다. `schemaVersion`만 올리면 안전할 거라 생각했던 게 첫 번째 실수였다.

## 주의할 점

Realm 파일만 바뀌고 Riverpod이나 GetX가 이전 값을 들고 있으면 화면이 잠깐 예전 상태를 보여줄 수 있다. 로컬 조회가 끝난 뒤 기기 목록, 선택된 Space, MQTT 구독 토픽을 함께 확인해야 한다.

요점만 남기면 이렇다.

- 새 설치 테스트와 이전 버전 fixture 테스트는 목적이 다르다.
- fixture는 매번 복사해 한 번만 마이그레이션되게 한다.
- 새 필드의 기본값과 기존 기기·방 이름 보존을 명시적으로 검증한다.
- Realm 변환 테스트와 MQTT 통신 테스트는 Fake 경계로 분리한다.

`Flutter integration_test`에서 Realm 마이그레이션을 한 번이라도 검증해 두면, 앱 업데이트 후 “기기 목록이 비었다”는 치명적인 회귀를 배포 전에 확인할 수 있다.
