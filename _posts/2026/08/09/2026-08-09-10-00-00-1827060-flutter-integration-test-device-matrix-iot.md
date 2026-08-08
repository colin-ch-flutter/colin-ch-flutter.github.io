---
layout: post
title: "Flutter integration_test 실기기 매트릭스 - Android·iOS IoT 테스트를 CI에서 나누는 법"
description: "Flutter integration_test를 Android·iOS 실기기 매트릭스로 나누고, BLE와 MQTT가 섞인 IoT 테스트에서 기기별 실패를 격리하는 GitHub Actions 전략을 정리했다."
date: 2026-08-09
tags: [Flutter, integration_test, Android, iOS, IoT, CI/CD]
comments: true
share: true
---

![Flutter integration_test Android iOS 실기기 매트릭스](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

이 그림에서 봐야 할 부분은 하나의 테스트 명령을 모든 기기에 던지는 대신, 플랫폼별 환경을 분리해 결과를 남기는 구조다.

`Flutter integration_test`를 에뮬레이터 하나에서 통과시켰는데도 실제 IoT 앱은 Android에서만 BLE 권한이 실패하고, iOS에서는 MQTT 복귀 시점에 멈췄다. 처음엔 테스트 코드가 틀렸다고 생각했는데, 해보니 문제는 테스트보다 **실행 기기와 권한 상태를 한 묶음으로 관리하지 않은 것**에 있었다.

## 기기별로 달라지는 실패 지점

| 환경 | 실제로 확인할 것 | 격리할 값 |
|---|---|---|
| Android 실기기 | BLE 스캔 권한, 백그라운드 복귀 | `ANDROID_SERIAL`, 권한 초기화 |
| iOS 실기기 | Bluetooth 권한, 앱 재설치 | `IOS_DEVICE_ID`, signing |
| 에뮬레이터 | 화면 전환, Fake MQTT | `TEST_MODE=fake` |

BLE를 쓰는 테스트는 시뮬레이터의 성공을 실기기 성공으로 보면 안 된다. 반대로 로그인·공간 전환 같은 화면 흐름까지 모두 실기기에서 돌리면 비용과 시간이 커진다. 테스트 목적에 따라 실행 대상을 나눠야 한다.

## 테스트에서 플랫폼을 명시한다

테스트 코드가 실행 환경을 직접 추측하지 않도록 `dart-define`으로 범위를 고정했다. 아래 값은 BLE 실기기 시나리오와 서버를 거치지 않는 smoke 시나리오를 구분한다.

```dart
const testPlatform = String.fromEnvironment('TEST_PLATFORM');
const useFakeTransport = bool.fromEnvironment(
  'USE_FAKE_TRANSPORT',
  defaultValue: false,
);

testWidgets('보일러 상태 카드를 표시한다', (tester) async {
  final app = createTestApp(
    platform: testPlatform,
    useFakeTransport: useFakeTransport,
  );

  await tester.pumpWidget(app);
  await tester.pump(const Duration(milliseconds: 300));

  expect(find.text('거실 보일러'), findsOneWidget);
});
```

`TEST_PLATFORM`은 화면에 표시할 문구가 아니라 Fake 서비스 선택과 로그 prefix를 결정하는 값이다. 테스트 안에서 `Platform.isAndroid`를 기준으로 분기하기 시작하면 로컬과 CI의 동작이 달라지기 쉽다.

## GitHub Actions 매트릭스로 나눈다

CI에서는 같은 테스트를 무작정 병렬 실행하지 않고, 실기기와 에뮬레이터의 역할을 매트릭스로 나눴다.

```yaml
strategy:
  fail-fast: false
  matrix:
    include:
      - name: android-smoke
        os: ubuntu-latest
        device: emulator-5554
        args: "--dart-define=TEST_PLATFORM=android --dart-define=USE_FAKE_TRANSPORT=true"
      - name: ios-widget-flow
        os: macos-latest
        device: "iPhone 15"
        args: "--dart-define=TEST_PLATFORM=ios --dart-define=USE_FAKE_TRANSPORT=true"

steps:
  - run: flutter test integration_test -d "${{ matrix.device }}" ${{ matrix.args }}
```

실제 BLE 테스트는 별도의 self-hosted runner에서 `device: android-ble-01`처럼 고정한다. 실기기는 여러 작업이 동시에 권한 팝업을 만지면 결과가 오염되므로 `max-parallel: 1`을 두는 편이 낫다. 테스트가 느려지는 건 감수해야 한다. 기기 공유로 생기는 재시도 비용이 더 컸다.

## 실패 로그도 플랫폼별로 보존한다

`fail-fast: false`를 넣은 이유는 Android 실패가 iOS 결과까지 가리지 않게 하려는 것이다. 각 작업에서 `build/integration_test/${{ matrix.name }}` 아래에 Flutter 로그, 스크린샷, `flutter doctor -v` 결과를 저장하면 재현 없이 원인을 좁힐 수 있다.

특히 BLE 권한 실패는 테스트 assertion보다 OS 로그가 더 직접적이다. 권한 팝업이 이미 처리된 기기인지, 앱이 삭제되지 않은 상태인지까지 함께 남겨야 한다.

## 운영하면서 정한 기준

- 에뮬레이터: 모든 PR에서 화면·상태 흐름 실행
- Android 실기기: BLE 스캔과 연결 smoke를 하루 1회 실행
- iOS 실기기: 권한과 백그라운드 복귀 regression을 배포 전 실행
- 실패 작업: 플랫폼별 로그와 스크린샷을 7일 보관

처음부터 모든 조합을 만들 필요는 없다. 테스트가 검증하려는 것이 화면인지, BLE 권한인지, MQTT 연결인지 나눈 뒤 가장 작은 매트릭스를 만들면 된다.

정리하면 `Flutter integration_test` 실기기 CI의 핵심은 기기 수를 늘리는 데 있지 않다. 플랫폼별로 다른 실패 조건을 분리하고, 각 실행에 동일한 설정·로그·초기화 규칙을 붙이는 데 있다.
