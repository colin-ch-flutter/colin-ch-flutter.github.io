---
layout: post
title: "Flutter integration_test 테스트 태그 - IoT smoke·regression을 CI에서 골라 실행하기"
description: "Flutter integration_test를 smoke·regression·device 태그로 나누고 GitHub Actions에서 필요한 IoT 시나리오만 선택 실행하는 방법을 정리했다."
date: 2026-08-08
tags: [Flutter, integration_test, IoT, CI/CD, CleanArchitecture]
comments: true
share: true
---

![Flutter integration_test 테스트 태그를 나눠 실행하는 IoT CI 전략](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

Flutter integration_test를 CI에서 빠르게 돌리려면 파일을 나누는 것만으로는 부족하다. IoT 앱의 보일러 제어 smoke 테스트와 BLE 권한·오프라인 회귀 테스트를 태그로 구분해야 PR에서는 핵심 흐름만, 배포 전에는 전체 시나리오만 실행할 수 있다. [실행 시간을 줄이기 위해 테스트 스위트를 분리한 방법]({% post_url 2026-08-08-10-00-00-1802370-flutter-integration-test-speed-optimization-iot %})에서 한 단계 더 나아간 방식이다.

## 파일보다 태그가 관리하기 편했던 이유

처음에는 `integration_test/smoke`, `integration_test/regression` 폴더만 만들었다. 그런데 같은 로그인·공간 선택 흐름이 여러 파일에 복사됐고, 특정 시나리오만 재실행할 때 경로를 기억해야 했다. 테스트가 20개를 넘으니 폴더 구조보다 실행 목적을 코드에 붙이는 편이 명확했다.

| 태그 | 포함할 흐름 | 실행 시점 |
|---|---|---|
| `smoke` | 로그인, 보일러 전원, MQTT 상태 반영 | 모든 PR |
| `regression` | 로그아웃, 오프라인 복구, 잘못된 명령 | main 머지 후 |
| `device` | BLE 권한, 화면 회전, 백그라운드 복귀 | 실기기 또는 야간 |

## Dart 테스트에 태그를 붙인다

`@Tags`는 테스트 이름을 바꾸지 않고도 실행 그룹을 추가한다. 아래처럼 한 파일 안에서 smoke와 regression을 분리할 수 있다.

```dart
import 'package:integration_test/integration_test.dart';
import 'package:flutter_test/flutter_test.dart';

import '../helpers/test_app.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('보일러 제어', () {
    testWidgets('연결된 공간에서 전원을 켠다', (tester) async {
      await launchTestApp(tester, scenario: TestScenario.connectedHome);
      await tester.tap(find.text('보일러'));
      await tester.tap(find.text('전원 켜기'));
      expect(find.text('가동 중'), findsOneWidget);
    }, tags: ['smoke']);

    testWidgets('오프라인에서 명령 실패를 표시한다', (tester) async {
      await launchTestApp(tester, scenario: TestScenario.offline);
      await tester.tap(find.text('전원 켜기'));
      expect(find.text('연결을 확인해 주세요'), findsOneWidget);
    }, tags: ['regression']);
  });
}
```

테스트 코드 바로 위에 시나리오를 설명하는 이유는 태그만 보고도 Fake 환경과 검증 범위를 판단하게 하려는 것이다. `connectedHome`은 BLE와 MQTT를 Fake로 대체하고, `offline`은 네트워크 응답을 끊은 상태다. 실제 기기 통신 자체를 검증하는 테스트에는 `device` 태그를 붙여 CI의 에뮬레이터에서 잘못 통과하지 않게 했다.

## GitHub Actions에서 선택 실행한다

PR과 main 브랜치의 실행 범위를 다르게 두면 실패 원인도 짧아진다. `--tags` 뒤에는 실행할 태그를 넣고, `--exclude-tags`로 실기기 전용 테스트를 제외한다.

```yaml
- name: Run integration smoke tests
  if: github.event_name == 'pull_request'
  run: flutter test integration_test -d emulator-5554 --tags smoke

- name: Run integration regression tests
  if: github.ref == 'refs/heads/master'
  run: flutter test integration_test -d emulator-5554 --tags 'smoke,regression' --exclude-tags device
```

여기서 주의할 점은 태그 필터가 테스트 격리를 대신하지 않는다는 것이다. 테스트마다 앱 상태와 Realm 데이터를 초기화하고, Fake MQTT 구독도 닫아야 한다. [네트워크를 실제 서버 없이 격리한 구성]({% post_url 2026-08-06-16-00-00-1752990-flutter-integration-test-network-isolation-iot %})을 적용하지 않으면 smoke만 실행해도 이전 테스트의 토픽이 남을 수 있다.

## 운영하면서 정한 기준

- PR smoke는 3분 안에 끝나야 한다.
- 실패한 태그만 로컬에서 `--tags`로 재현한다.
- `device` 테스트가 실패해도 smoke 성공으로 배포하지 않는다.
- 같은 테스트에 태그를 네 개 이상 붙이지 않는다. 분류 기준이 흐려진다.

처음엔 전체 테스트를 매번 돌리는 것이 안전하다고 생각했다. 해보니 실패 로그가 너무 많아 실제 회귀를 놓쳤다. 태그는 테스트를 덜 하는 장치가 아니라, 어떤 품질 신호를 언제 확인할지 명시하는 경계였다.

짧게 정리하면 `smoke`는 빠른 배포 판단, `regression`은 기능 회귀, `device`는 네이티브 차이 검증이다. 세 그룹을 같은 명령으로 묶지 않는 것만으로도 Flutter IoT integration_test CI가 훨씬 읽기 쉬워진다.
