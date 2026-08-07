---
layout: post
title: "Flutter integration_test 실행 시간 줄이기 - IoT 테스트 스위트 분리 전략"
description: "Flutter integration_test가 CI에서 오래 걸리는 원인을 IoT 앱의 앱 재시작·실제 네트워크·중복 로그인 흐름으로 나눠 보고, smoke와 regression 테스트를 분리해 실행 시간을 줄이는 방법을 정리했다."
date: 2026-08-08
tags: [Flutter, integration_test, IoT, CI/CD, CleanArchitecture]
comments: true
share: true
---

![Flutter integration_test 실행 시간을 줄이는 IoT 테스트 전략](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

Flutter integration_test가 CI에서 10분 넘게 걸린다면 테스트 하나하나를 더 빠르게 만드는 것보다 **실행할 테스트를 나누는 것**이 먼저다. IoT 앱에서는 로그인, 공간 선택, BLE Fake 연결, MQTT 상태 동기화가 모든 테스트에 반복되기 쉽다. 나도 처음엔 `Future.delayed`를 줄이고 `pumpAndSettle`을 덜 호출했는데, 전체 시간은 거의 줄지 않았다.

## 오래 걸리는 이유는 앱 시작에 있다

최근 테스트를 재보니 화면 검증 자체는 2~3초인데, 앱을 새로 띄우고 테스트 계정을 준비하는 시간이 매번 8초 안팎이었다. 12개 시나리오를 순서대로 실행하면 실제 검증보다 준비 과정에 더 많은 시간이 들어간다.

| 구분 | 실행 대상 | 목적 | 실행 빈도 |
|---|---|---|---|
| Smoke | 로그인·보일러 제어 2~3개 | 배포 가능 여부 확인 | PR마다 |
| Regression | 권한·오프라인·실패 복구 | 전체 회귀 확인 | main 머지 후 |
| Device | OS·화면·실기기 조합 | 플랫폼 차이 확인 | 야간 또는 릴리스 |

이렇게 나누면 PR에서 3분짜리 핵심 흐름만 확인하고, 15분 가까이 걸리는 전체 검증은 별도 파이프라인으로 보낼 수 있다.

## 테스트 그룹을 코드에서 분리한다

테스트 파일을 무작정 여러 개로 쪼개는 것보다 실행 대상 목록을 명시하는 편이 관리하기 쉬웠다. `integration_test/smoke/`에는 실제 장애가 나면 바로 알림이 필요한 흐름만 둔다.

```dart
// integration_test/smoke/boiler_control_test.dart
import 'package:integration_test/integration_test.dart';
import 'package:flutter_test/flutter_test.dart';

import '../helpers/test_app.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('보일러 온도 변경 smoke', (tester) async {
    await launchTestApp(tester, scenario: TestScenario.connectedHome);

    await tester.tap(find.text('보일러'));
    await tester.tap(find.text('온도 올리기'));
    await tester.pump(const Duration(milliseconds: 300));

    expect(find.text('23°C'), findsOneWidget);
  });
}
```

코드 위의 `launchTestApp`은 로그인과 Fake 서비스 주입을 한 곳에서 처리한다. 테스트마다 같은 초기화 코드를 복사하지 않으니, smoke를 3개로 늘려도 준비 로직이 세 군데로 퍼지지 않는다. 실제 네트워크는 이 계층에서 끊고, 서버까지 확인해야 하는 테스트만 별도 suite로 남겼다. [Flutter integration_test 네트워크 격리]({% post_url 2026-08-06-16-00-00-1752990-flutter-integration-test-network-isolation-iot %})에서 정리한 Fake Repository 구조가 이 분리에 그대로 맞았다.

## CI에서는 명령어 자체를 나눈다

```yaml
# .github/workflows/integration-test.yml
- name: Run smoke tests
  run: flutter test integration_test/smoke -d emulator-5554

- name: Run regression tests
  if: github.ref == 'refs/heads/master'
  run: flutter test integration_test/regression -d emulator-5554
```

PR에서 regression을 완전히 빼면 위험하지 않냐는 생각이 들 수 있다. 그래서 smoke에는 로그인 성공만 넣지 않고, 앱의 핵심 가치가 실제로 동작하는 “기기 목록 표시 → 보일러 제어 → 상태 카드 갱신”을 넣었다. 실패 로그와 스크린샷은 두 suite 모두 남겨야 한다. [CI에서 integration_test 실패 원인 남기기]({% post_url 2026-08-07-16-00-00-1790025-flutter-integration-test-ci-report-iot %})처럼 실행 시간 단축과 진단 자료 수집은 별개 문제다.

## 주의할 점

`testWidgets` 여러 개가 같은 앱 인스턴스를 공유한다고 가정하면 안 된다. 전역 Provider, Realm in-memory DB, 선택된 Space가 남아 순서 의존성이 생길 수 있다. 반대로 매 테스트마다 무거운 로그인과 앱 부팅을 넣으면 시간이 폭증한다. 해결 기준은 간단하다.

- 테스트 간 상태를 공유하지 않는다.
- 초기화는 Fake fixture로 빠르게 처리한다.
- 앱을 다시 띄우는 비용은 smoke가 아닌 공통 helper에서 측정한다.
- 속도 때문에 실제 네트워크 검증을 전부 없애지 않는다.

실제로 앱 부팅을 12번 하던 구성을 suite별로 나누고 네트워크를 Fake로 바꾸니 PR 검증은 9분대에서 2분 40초 정도로 줄었다. 테스트가 빨라진 덕분에 실패를 재현하기 위해 실행을 미루는 습관도 사라졌다.

짧게 정리하면, Flutter integration_test 최적화의 핵심은 delay 숫자를 줄이는 일이 아니다. IoT 앱에서 반드시 빠르게 확인할 흐름과 비용을 감수하고 전체 검증할 흐름을 분리하고, 각 suite의 목적을 CI 조건에 연결하는 일이다.
