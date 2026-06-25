---
layout: post
title: "Flutter Integration Test로 IoT 앱 E2E 검증하기 — integration_test 실전 설정"
description: "unit·widget·golden 테스트 다음 단계, Flutter integration_test로 BLE 연결부터 기기 제어까지 실제 앱 흐름을 통째로 검증하는 방법을 정리했다. 타이밍 이슈로 flaky 해지는 원인과 CI 실행 전략도 포함."
date: 2026-06-25
tags: [Flutter, integration_test, IoT, CI/CD, Android]
comments: true
share: true
---

# Flutter Integration Test로 IoT 앱 E2E 검증하기 — integration_test 실전 설정

![Flutter integration test E2E 앱 자동화 테스트 실행 화면](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

unit test로 Repository를 검증하고, widget test로 제어 화면을 확인하고, golden test로 UI 스냅샷까지 찍었는데도 "근데 실제로 앱 켜보면 왜 이상하지?"라는 상황이 생긴다. 그게 integration test가 필요한 이유다. Flutter integration_test 패키지를 IoT 앱에 붙여본 경험을 그대로 정리했다.

## integration_test가 unit/widget test와 다른 점

unit test는 클래스 하나 잘라내서 검증한다. widget test는 화면 렌더링과 버튼 탭까지 확인하지만 실제 OS 위에서 돌지는 않는다. integration test는 **실제 에뮬레이터나 실기기 위에서 앱 전체를 띄워서** 처음부터 끝까지 흐름을 검증한다.

우리 IoT 앱 기준으로 하면:

| 테스트 종류 | 검증 범위 | 실행 속도 |
|------------|----------|---------|
| unit | Repository, Controller 로직 | 빠름 (수백ms) |
| widget | 화면 렌더링, 탭 반응 | 중간 (수초) |
| golden | UI 스냅샷 픽셀 비교 | 중간 |
| integration | 로그인 → 기기 목록 → 제어 전체 흐름 | 느림 (수십초) |

느린 건 맞다. 그래서 전체 CI에서 매 커밋마다 돌리는 게 아니라, PR merge 전이나 nightly build에 넣는 게 현실적이다.

## 패키지 설정

```yaml
# pubspec.yaml
dev_dependencies:
  integration_test:
    sdk: flutter
  flutter_test:
    sdk: flutter
```

`integration_test`는 Flutter SDK에 포함돼 있어서 별도 버전 지정 없이 `sdk: flutter`만 쓰면 된다. Flutter 3.24 기준으로 안정적으로 동작한다.

디렉터리 구조는 이렇게 잡았다:

```
my_iot_app/
├── lib/
│   └── main.dart
├── integration_test/
│   ├── app_test.dart          ← 메인 E2E 시나리오
│   └── helpers/
│       ├── app_runner.dart    ← 앱 초기화 헬퍼
│       └── fake_ble_driver.dart  ← 실기기 없을 때 Fake 주입
├── test_driver/
│   └── integration_test.dart  ← flutter drive용 드라이버 (선택)
```

## app_runner.dart — 테스트용 앱 진입점 분리

처음에 그냥 `main.dart`를 그대로 불렀더니 Firebase 초기화에서 터졌다. 테스트 환경에서는 실제 Firebase 대신 mock을 주입해야 해서, 앱 진입점을 별도로 뺐다.

```dart
// integration_test/helpers/app_runner.dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';
import 'package:my_iot_app/core/di/injection.dart';
import 'package:my_iot_app/main_widget.dart';

Future<void> startApp({bool useFakeBle = true}) async {
  // Firebase 대신 Fake 서비스로 DI 교체
  await setupTestDependencies(useFakeBle: useFakeBle);
  runApp(const MyIoTApp());
}
```

`setupTestDependencies`에서 GetX DI를 통해 BLE 서비스와 MQTT 서비스를 Fake로 교체한다. [이전 글에서 BLE Fake 서비스 패턴을 다뤘다]({% post_url 2026-06-24-10-00-00-654285-flutter-blue-plus-ble-unit-test-wrapper-isolation %}).

## 첫 번째 integration test 작성

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'helpers/app_runner.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('홈 화면 E2E', () {
    testWidgets('로그인 후 기기 목록이 표시된다', (tester) async {
      await startApp(useFakeBle: true);
      await tester.pumpAndSettle();

      // 로그인 화면 확인
      expect(find.text('로그인'), findsOneWidget);

      // 이메일·비밀번호 입력
      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'testPassword123',
      );
      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle(const Duration(seconds: 3));

      // 홈 화면으로 이동 확인
      expect(find.text('내 기기'), findsOneWidget);
    });

    testWidgets('보일러 카드 탭 → 제어 화면 진입', (tester) async {
      await startApp(useFakeBle: true);
      await tester.pumpAndSettle();

      // 로그인 (헬퍼로 분리하면 더 깔끔)
      await _doLogin(tester);

      // 보일러 카드 탭
      await tester.tap(find.byKey(const Key('device_card_boiler_01')));
      await tester.pumpAndSettle();

      expect(find.text('보일러 제어'), findsOneWidget);
      expect(find.byKey(const Key('power_toggle')), findsOneWidget);
    });
  });
}

Future<void> _doLogin(WidgetTester tester) async {
  await tester.enterText(find.byKey(const Key('email_field')), 'test@example.com');
  await tester.enterText(find.byKey(const Key('password_field')), 'testPassword123');
  await tester.tap(find.byKey(const Key('login_button')));
  await tester.pumpAndSettle(const Duration(seconds: 3));
}
```

`IntegrationTestWidgetsFlutterBinding.ensureInitialized()` — 이걸 빠트리면 테스트가 아예 안 돌아간다. widget test의 `TestWidgetsFlutterBinding`과는 다른 바인딩이다.

## 실행 방법

```bash
# 에뮬레이터나 실기기 연결 후
flutter test integration_test/app_test.dart -d emulator-5554

# 특정 테스트만
flutter test integration_test/app_test.dart \
  --name "보일러 카드 탭" \
  -d emulator-5554
```

`flutter run`이 아니라 `flutter test`다. 이게 헷갈려서 처음에 한참 삽질했다. `flutter drive`는 이제 레거시 방식이라 `flutter test`로 통일하는 게 맞다.

## pumpAndSettle이 타임아웃 나는 문제

IoT 앱 특성상 MQTT 연결이나 BLE 스캔이 끝날 때까지 기다려야 해서 `pumpAndSettle`을 남발했더니 계속 타임아웃이 났다.

```
TimeoutException after 0:00:10.000000: Test timed out after 10 seconds.
```

`pumpAndSettle`은 내부적으로 "더 이상 프레임이 없을 때까지" 기다리는데, 백그라운드에서 MQTT가 계속 heartbeat를 보내면 영원히 끝나지 않는다.

해결책은 두 가지다:

```dart
// 1. 타임아웃을 명시적으로 늘린다
await tester.pumpAndSettle(const Duration(seconds: 10));

// 2. pumpAndSettle 대신 pump를 직접 제어한다
await tester.pump(const Duration(seconds: 2));
await tester.pump(const Duration(seconds: 2));
// 이렇게 하면 프레임을 강제로 2초씩 진행
```

MQTT Fake 서비스에서 heartbeat 타이머를 꺼두는 게 근본 해결책이었다. 테스트용 Fake에서는 주기적 이벤트를 모두 비활성화하고, 필요한 이벤트는 테스트 코드에서 수동으로 emit하도록 바꿨다.

## 실기기 vs 에뮬레이터

BLE 관련 통합 테스트는 에뮬레이터에서 불가능하다. iOS 시뮬레이터도 마찬가지.

| 기능 | 에뮬레이터 | 실기기 |
|------|----------|------|
| UI 흐름 | O | O |
| HTTP API 호출 | O | O |
| BLE 스캔·연결 | X | O |
| Firebase FCM | 에뮬레이터 일부 O | O |

그래서 테스트를 두 그룹으로 나눴다:
- `app_test.dart`: 에뮬레이터에서 돌릴 수 있는 UI 흐름 (Fake BLE)
- `app_ble_test.dart`: 실기기 전용 (실제 BLE 연결 필요)

## GitHub Actions CI 설정

에뮬레이터 기반 integration test는 CI에 넣을 수 있다.

```yaml
# .github/workflows/integration_test.yml
name: Integration Test

on:
  pull_request:
    branches: [master]

jobs:
  integration-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'

      - name: Enable KVM (Android 에뮬레이터 가속)
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm

      - name: Run integration tests on emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          arch: x86_64
          script: flutter test integration_test/app_test.dart -d emulator-5554
```

`reactivecircus/android-emulator-runner` 없이 직접 에뮬레이터를 띄우려다 한참 고생했다. 이 액션이 AVD 생성부터 부팅 대기까지 알아서 처리해준다.

KVM 활성화는 빼먹으면 에뮬레이터가 정말 느려서 타임아웃이 난다. ubuntu-latest에서는 KVM이 기본으로 꺼져 있다.

![Flutter CI integration test GitHub Actions 실행 결과](https://images.unsplash.com/photo-1555949963-ff9fe0c870eb?w=800&q=80)

## 실제로 잡은 버그

unit test와 widget test를 다 통과했는데 integration test에서 처음 발견한 버그가 있었다.

로그인 후 홈 화면으로 Navigate하는 타이밍에, GetX의 라우팅이 완료되기 전에 기기 목록 API를 먼저 호출해서 `null` 접근 오류가 났다. widget test에서는 `pumpAndSettle`이 모든 프레임을 소화하기 때문에 이 타이밍 문제가 드러나지 않았던 거다.

실제 앱 흐름에서만 재현되는 종류의 버그를 잡으려면 integration test가 필요하다는 걸 직접 경험했다.

---

다음 편은 Firebase Test Lab을 이용해서 여러 실기기에서 동시에 integration test를 돌리는 내용이다.

**참고 링크**
- [Flutter 공식 Integration Test 문서](https://docs.flutter.dev/testing/integration-tests)
- [reactivecircus/android-emulator-runner](https://github.com/ReactiveCircus/android-emulator-runner)
- [Firebase Test Lab Flutter 가이드](https://firebase.google.com/docs/test-lab/flutter/integration-testing-with-flutter)
