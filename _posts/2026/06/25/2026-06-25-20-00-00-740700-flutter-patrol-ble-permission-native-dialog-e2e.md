---
layout: post
title: "Flutter Patrol — BLE 권한 다이얼로그까지 E2E 자동화하기"
description: "Flutter integration_test만으로는 Android 블루투스 권한 팝업과 iOS 알림 허용 다이얼로그를 제어할 수 없다. Patrol이 이 한계를 어떻게 해결하는지, IoT 앱의 BLE 권한 요청부터 기기 제어 흐름까지 실전 코드로 정리했다."
date: 2026-06-25
tags: [Flutter, Patrol, BLE, integration_test, Android, iOS, CI/CD]
comments: true
share: true
---

# Flutter Patrol — BLE 권한 다이얼로그까지 E2E 자동화하기

![Flutter Patrol 모바일 앱 E2E 자동화 테스트 실행](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?w=800&q=80)

[지난 편에서 integration_test로 IoT 앱 E2E 흐름을 검증했다.]({% post_url 2026-06-25-18-00-00-728355-flutter-integration-test-iot-e2e %}) 근데 BLE 앱에서 치명적인 문제가 있다. Android에서 앱을 처음 실행하면 "블루투스 주변 기기 검색 허용?" 팝업이 뜨는데, `integration_test`는 이 **네이티브 OS 다이얼로그를 건드릴 수 없다.** Patrol은 이 한계를 깔끔하게 해결한다.

## integration_test가 막히는 지점

이게 정확히 어디서 막히는지 먼저 설명하면:

```dart
// integration_test — 이렇게 써도 권한 팝업은 처리 못함
testWidgets('BLE 스캔 시작', (tester) async {
  await tester.tap(find.byKey(Key('scan_button')));
  await tester.pumpAndSettle();
  // Android 12+ 에서 BLUETOOTH_SCAN 권한 팝업이 뜨는 순간 테스트 멈춤
  // tester는 Flutter UI만 제어하고, OS 다이얼로그는 Flutter 밖에 있음
});
```

`tester`는 Flutter 렌더링 트리 안에서만 동작한다. OS가 띄운 시스템 팝업은 Flutter 밖이라 건드릴 수가 없다. 처음엔 `permission_handler`로 앱 시작 시 미리 권한 요청 하면 되지 않나 생각했는데, CI 에뮬레이터에서는 매번 앱이 새로 설치되고 권한도 초기화된다. 피할 방법이 없다.

## Patrol이 다른 이유

Patrol은 내부적으로 Android에서는 `UIAutomator2`, iOS에서는 `XCUITest`를 구동한다. Flutter 앱 위에 네이티브 자동화 레이어를 얹는 구조라서 OS 팝업도 제어할 수 있다.

```
integration_test: Flutter UI만 제어
Patrol:           Flutter UI + 네이티브 OS 다이얼로그 + 알림 트레이 + 딥링크
```

패키지는 두 개가 필요하다.

```yaml
# pubspec.yaml
dev_dependencies:
  patrol: ^3.15.0
  patrol_cli: ^3.15.0  # CLI 도구 — pub global activate patrol_cli
```

그리고 CLI 도구를 전역 설치한다.

```bash
dart pub global activate patrol_cli
```

## 플랫폼 설정 — Android

Patrol은 `integration_test`와 달리 네이티브 앱 파일 수정이 필요하다. Android는 `MainActivityTest.kt`를 만들어야 한다.

```kotlin
// android/app/src/androidTest/kotlin/com/example/yourapp/MainActivityTest.kt
package com.example.yourapp

import pl.leancode.patrol.PatrolJUnitRunner
import org.junit.runner.RunWith

@RunWith(PatrolJUnitRunner::class)
class MainActivityTest : PatrolTestActivity()
```

`android/app/build.gradle`에도 testInstrumentationRunner를 바꿔야 한다.

```groovy
android {
  defaultConfig {
    testInstrumentationRunner "pl.leancode.patrol.PatrolJUnitRunner"
  }
}
```

## 플랫폼 설정 — iOS

iOS는 `RunnerUITests` 타겟을 추가해야 한다. Xcode에서 `File > New > Target > UI Testing Bundle`로 만든 다음 아래처럼 수정한다.

```swift
// ios/RunnerUITests/RunnerUITests.swift
import XCTest
import patrol

@objc class RunnerUITests: PatrolTestCase {}
```

근데 솔직히 iOS 설정이 더 까다롭다. `RunnerUITests.xctestplan` 파일과 scheme 설정이 맞아야 하는데, 공식 문서의 `patrol bootstrap` 명령을 쓰면 자동으로 잡아준다.

```bash
patrol bootstrap
```

이 명령 하나로 Android/iOS 양쪽 네이티브 설정을 대부분 처리해준다. 수동으로 건드리기 전에 이거 먼저 돌려볼 것.

## BLE 권한 다이얼로그 자동화

드디어 핵심이다. Patrol에서는 `$` 또는 `PatrolIntegrationTester`를 쓴다.

```dart
// integration_test/ble_permission_test.dart
import 'package:patrol/patrol.dart';

void main() {
  patrolTest(
    'BLE 권한 허용 후 스캔 시작',
    ($) async {
      await $.pumpWidgetAndSettle(MyApp());

      // 스캔 버튼 탭 — 여기서 Android 권한 팝업이 뜸
      await $('스캔 시작').tap();

      // Android: 블루투스 권한 팝업 처리
      // 이게 integration_test에서 불가능했던 부분
      if (await $.native.isPermissionDialogVisible()) {
        await $.native.grantPermissionWhenInUse();
      }

      // 스캔 결과 화면 확인
      await $.waitUntilVisible(find.byKey(Key('device_list')));
      expect(find.byType(DeviceListTile), findsWidgets);
    },
  );
}
```

`$.native`가 네이티브 레이어 접근 포인트다. 여기에 권한 팝업 처리, 알림 트레이 열기, 딥링크 실행 같은 것들이 들어있다.

## IoT 앱 전체 흐름 테스트

BLE 권한 → 디바이스 스캔 → 연결 → 제어 명령 전송까지 하나의 테스트로 묶으면 이렇게 된다.

```dart
patrolTest(
  'IoT 기기 연결 후 보일러 제어',
  ($) async {
    await $.pumpWidgetAndSettle(MyApp());

    // 로그인 (저장된 계정이 있으면 자동 로그인 화면 통과)
    await $.waitUntilVisible(find.byKey(Key('home_screen')));

    // BLE 스캔 탭
    await $('기기 추가').tap();

    // 권한 팝업 — Android 12+ BLUETOOTH_SCAN, BLUETOOTH_CONNECT
    if (await $.native.isPermissionDialogVisible()) {
      await $.native.grantPermissionWhenInUse();
    }
    // 위치 권한 팝업이 추가로 뜨는 경우 (Android 11 이하)
    if (await $.native.isPermissionDialogVisible()) {
      await $.native.grantPermissionWhenInUse();
    }

    // 스캔 결과에서 테스트 기기 선택
    await $.waitUntilVisible(find.text('Colin-Boiler-001'));
    await $(find.text('Colin-Boiler-001')).tap();

    // 연결 대기 — BLE 연결은 2~5초 걸림
    await $.waitUntilVisible(
      find.byKey(Key('control_screen')),
      timeout: Duration(seconds: 10),
    );

    // 온도 설정
    await $(find.byKey(Key('temp_up'))).tap();
    await $(find.byKey(Key('temp_up'))).tap();
    expect(find.text('22°C'), findsOneWidget);

    // 전원 ON
    await $(find.byKey(Key('power_button'))).tap();
    await $.waitUntilVisible(find.byKey(Key('status_on')));
  },
);
```

실제 BLE 기기가 없는 CI 환경에서는 이 테스트가 `Colin-Boiler-001` 기기를 못 찾아서 실패한다. 그래서 두 가지 전략 중 하나를 선택해야 한다.

1. **에뮬레이터 + Mock BLE**: integration_test처럼 FakeBleService 주입 (권한 팝업 테스트는 못 함)
2. **실기기 Farm**: Firebase Test Lab 또는 BrowserStack에서 실기기로 실행

## CI에서 Patrol 실행

Patrol은 `flutter test`가 아니라 `patrol test`로 실행한다.

```bash
# Android 에뮬레이터에서 실행
patrol test --target integration_test/ble_permission_test.dart

# iOS 시뮬레이터에서 실행
patrol test --target integration_test/ble_permission_test.dart --device "iPhone 16"
```

GitHub Actions에서 쓰려면:

```yaml
# .github/workflows/patrol.yml
- name: Install Patrol CLI
  run: dart pub global activate patrol_cli

- name: Run Patrol tests (Android)
  uses: reactivecircus/android-emulator-runner@v2
  with:
    api-level: 33
    script: patrol test --target integration_test/ble_permission_test.dart
```

**Firebase Test Lab 주의:** 2026년 중반 기준으로 Firebase Test Lab에서 Patrol 지원이 불완전하다. `patrol_cli`의 `--verbose` 로그를 보면 `PatrolJUnitRunner` 연결 단계에서 타임아웃이 나는 경우가 있다. BrowserStack Device Cloud나 LambdaTest가 호환성이 더 좋다.

![Patrol 테스트 실행 결과 CI 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

## 삽질한 포인트

**1. `patrol bootstrap` 이후 Xcode scheme이 안 잡히는 경우**

`ios/Runner.xcworkspace`를 Xcode에서 열어서 `Product > Scheme > Manage Schemes`에서 `RunnerUITests` 가 있는지 확인한다. 없으면 Target을 수동으로 추가해야 한다. `patrol bootstrap`이 항상 완벽하진 않다.

**2. Android `isPermissionDialogVisible()` false 반환**

Android API 33+ 에서 권한 팝업이 `ActivityCompat.requestPermissions`가 아닌 시스템 앱으로 뜨는 경우 Patrol이 못 잡을 때가 있다. `$.native.grantPermissionWhenInUse()` 대신 `$.native.tapOnAndroidDialog()` 로 버튼 텍스트 직접 탭하는 방법으로 우회했다.

```dart
// 버튼 텍스트로 직접 탭
await $.native.tap(Selector(text: '앱 사용 중에만 허용'));
```

**3. iOS BLE 권한 팝업**

iOS에서 BLE 권한은 `CoreBluetooth`가 처음 접근할 때 뜬다. Patrol에서는 `$.native.grantPermissionWhenInUse()`로 잡히는데, 타이밍이 중요하다. `pumpAndSettle` 이후에 권한 팝업 체크를 해야 한다. 너무 빠르면 팝업이 아직 안 뜬 상태라 `isPermissionDialogVisible()`이 false를 반환한다.

---

[BLE 연결 안정성 문제]({% post_url 2026-05-11-09-24-36-148140-ble-connection-stability-reconnect-error-handling %})는 unit test + Patrol 조합으로 레이어별로 검증하는 게 가장 신뢰도 높다. unit test로 재연결 로직을 검증하고, Patrol로 실기기에서 OS 권한까지 포함한 전체 흐름을 확인한다.

다음 편은 테스트 커버리지 100% 집착에서 벗어나 **"어떤 코드에 테스트가 없으면 위험한가"** 를 기준으로 삼는 테스트 전략 정리다.

**참고 링크**
- [Patrol 공식 문서](https://patrol.leancode.co/)
- [patrol pub.dev](https://pub.dev/packages/patrol)
- [Patrol GitHub](https://github.com/leancodepl/patrol)
