---
layout: post
title: "Flutter integration_test BLE 권한 복구 - 설정 이동 후 재스캔 검증"
description: "Flutter integration_test에서 BLE 권한을 영구 거부한 뒤 설정 화면으로 이동하고, 앱 복귀 후 flutter_blue_plus 재스캔이 정상 동작하는지 검증하는 방법을 정리했다."
date: 2026-08-25
tags: [Flutter, Dart, BLE, flutter_blue_plus, Android, iOS, IoT, 스마트홈]
comments: true
share: true
---

![Flutter integration_test BLE 권한 거부 후 설정 복귀와 재스캔](/assets/images/flutter-integration-test-ble-permission-recovery-iot.png)

Flutter integration_test에서 BLE 권한을 한 번 허용하는 흐름만 테스트하면 실사용 버그를 놓친다. 사용자가 권한을 영구 거부한 뒤 설정으로 이동하고, 다시 앱으로 돌아와 `flutter_blue_plus` 스캔을 재시작하는 경로까지 검증해야 한다. 처음엔 `request()`가 false를 반환하는지만 보면 된다고 생각했는데, 해보니 앱 복귀 뒤에도 이전의 `permissionDenied` 상태가 남아 스캔 버튼이 계속 비활성화됐다.

## 권한 거부와 스캔 실패는 다르다

BLE 등록 화면에서는 아래 상태를 분리해야 테스트 assertion도 명확해진다.

| 상태 | 화면에 보여줄 것 | 다음 동작 |
|---|---|---|
| 권한 요청 전 | 블루투스 권한 필요 | 권한 요청 |
| 일시 거부 | 권한이 필요함 | 다시 요청 |
| 영구 거부 | 설정에서 권한을 허용해야 함 | 설정 열기 |
| 허용됨 | 주변 기기 검색 가능 | 스캔 시작 |
| 허용됐지만 결과 없음 | 주변 기기를 찾지 못함 | 재스캔 |

`permanentlyDenied`를 단순한 스캔 실패로 처리하면 사용자는 계속 같은 버튼만 누르게 된다. Repository는 권한 상태를 반환하고, Controller는 영구 거부일 때 설정 이동 이벤트를 내보내도록 나눴다.

```dart
Future<void> startScan() async {
  final status = await blePermission.status();

  if (status == BlePermission.permanentlyDenied) {
    state = BleScreenState.openSettings;
    return;
  }
  if (status != BlePermission.granted) {
    state = BleScreenState.permissionRequired;
    return;
  }

  state = BleScreenState.scanning;
  await ble.scan(timeout: const Duration(seconds: 8));
}
```

코드 위에서 권한을 먼저 확인하는 이유는 권한이 없는 상태에서 스캔 API를 호출하면 Android와 iOS의 실패 방식이 달라지기 때문이다. `flutter_blue_plus`의 빈 결과를 권한 문제로 오해하지 않게 경계를 세웠다.

## integration_test에서 복귀 흐름 만들기

실제 시스템 설정 앱을 매번 자동화하지 않고, 앱 안에 테스트 빌드 전용 `debug-open-settings` 버튼과 복귀 트리거를 두었다. 설정 앱 자체를 검증하는 게 아니라 우리 앱이 복귀 이벤트를 받아 권한을 다시 조회하는지를 보는 테스트다.

```dart
testWidgets('BLE 권한 설정 복귀 후 재스캔한다', (tester) async {
  final fakePermission = FakeBlePermission(
    statuses: [
      BlePermission.permanentlyDenied,
      BlePermission.granted,
    ],
  );
  final fakeBle = FakeBleService();

  await launchTestApp(
    tester,
    permission: fakePermission,
    ble: fakeBle,
  );

  await tester.tap(find.byKey(const Key('scan-button')));
  expect(find.text('설정에서 권한을 허용해 주세요'), findsOneWidget);

  await tester.tap(find.byKey(const Key('open-settings-button')));
  await tester.tap(find.byKey(const Key('debug-resume-button')));
  await tester.tap(find.byKey(const Key('scan-button')));

  expect(fakePermission.statusCalls, 2);
  expect(fakeBle.scanCalls, 1);
  expect(find.text('주변 기기 검색 중'), findsOneWidget);
});
```

여기서 중요한 assertion은 설정 버튼이 보였다는 사실보다 복귀 뒤 `status()`를 다시 호출했는지다. 권한 조회를 `initState()`에서 한 번만 하면 설정 앱에서 허용한 결과가 반영되지 않는다. 실제 테스트에서는 `resumed` 이벤트를 받은 뒤 권한 상태를 갱신하고, 허용된 경우에만 스캔을 시작하게 했다.

## 실기기에서 확인할 범위

Fake 테스트는 상태 전이와 호출 순서를 보장한다. OS 권한 화면까지 확인하려면 별도 Patrol 테스트를 Android·iOS 실기기에서 실행해야 한다. 특히 Android는 “다시 묻지 않음”을 선택한 뒤 영구 거부가 되는 시점이 버전과 기존 권한 이력에 따라 달라진다. iOS도 앱 삭제 후 첫 실행인지에 따라 팝업이 다시 뜨지 않을 수 있다.

실패 로그에는 `permission status`, `resumed 시각`, `scanCalls`를 함께 남겼다. 화면에 권한 안내가 떴는데 스캔이 호출됐다면 Controller 분기 문제고, 복귀 후 status가 갱신되지 않았다면 생명주기 연결 문제다. 둘을 구분하지 않으면 권한 팝업만 반복해서 만지게 된다.

솔직하게 정리하면 BLE 권한 테스트의 핵심은 “허용 성공”이 아니다. 거부 → 설정 이동 → 앱 복귀 → 권한 재조회 → 재스캔이라는 상태 전이를 끊김 없이 검증하는 데 있다. `flutter_blue_plus` 결과가 비어 있을 때 권한 문제와 주변 기기 없음도 별도 상태로 남겨야 Flutter 스마트홈 등록 화면이 실제 사용자 상황을 제대로 설명할 수 있다.
