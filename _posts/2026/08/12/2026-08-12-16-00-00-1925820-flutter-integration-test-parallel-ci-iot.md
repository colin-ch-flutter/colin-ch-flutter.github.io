---
layout: post
title: "Flutter integration_test 병렬 실행 - CI 테스트 시간을 줄인 IoT 앱 전략"
description: "Flutter integration_test가 CI에서 오래 걸리는 문제를 테스트 그룹 분리와 병렬 실행으로 줄이는 방법을 정리했다. MQTT·FCM·딥링크 E2E의 격리 기준과 실패 재시도 함정도 포함했다."
date: 2026-08-12
tags: [Flutter, Dart, integration_test, CI/CD, IoT, MQTT, FCM, Android]
comments: true
share: true
---

![Flutter integration_test 병렬 실행과 IoT CI 테스트 파이프라인](/assets/images/flutter-integration-test-parallel-ci-iot.png)

이 그림에서 봐야 할 부분은 테스트 기기를 많이 띄우는 장면이 아니라, 서로 상태를 공유하지 않는 테스트 묶음을 여러 실행 레인으로 나누는 흐름이다.

Flutter integration_test가 늘어나면 테스트 실패보다 CI 대기 시간이 먼저 문제가 된다. 내 IoT 앱은 MQTT 재연결, FCM 알림, 딥링크, JWT 인증까지 붙인 뒤 Android 에뮬레이터 한 대에서 28분을 기다려야 했다. 처음엔 `--concurrency` 옵션 하나면 빨라질 줄 알았는데, integration_test는 앱 프로세스와 테스트 계정 상태를 공유해서 단순 병렬화가 위험했다. 해보니 테스트를 성격별로 나누고, 각 워커에 별도 계정을 주는 쪽이 안정적이었다.

## 테스트 파일을 기능이 아니라 상태 경계로 나눈다

파일 이름만 보고 `auth_test`, `mqtt_test`처럼 나누면 같은 Space와 MQTT 토픽을 건드리는 테스트가 서로 충돌한다. 기준은 기능보다 외부 상태다.

| 그룹 | 포함한 흐름 | 공유하면 생기는 문제 |
|---|---|---|
| smoke | 앱 시작, 로그인, 홈 화면 | 비교적 안전하지만 가장 먼저 실행 |
| device | 보일러 켜기, 상태 동기화 | MQTT retained 메시지와 기기 상태 충돌 |
| notification | FCM intent, 딥링크 | 알림 큐와 앱 생명주기 충돌 |
| recovery | 오프라인, 재연결, 토큰 만료 | 실행 시간이 길고 재시도 오염 가능 |

Smoke는 모든 커밋에서 돌리고, device·notification·recovery는 병렬 워커에 배치했다. [Flutter integration_test 실패 로그와 스크린샷 수집]({% post_url 2026-08-07-16-00-00-1790025-flutter-integration-test-ci-report-iot %})도 그룹별 아티팩트 경로를 나누니 어느 워커에서 깨졌는지 바로 확인할 수 있었다.

## 테스트 그룹을 실행 인자로 받는다

앱이 어떤 테스트를 실행할지 `dart-define`으로 결정하면 같은 빌드 설정을 유지하면서 CI 매트릭스만 늘릴 수 있다. 아래처럼 그룹별 진입점을 둔다.

```dart
enum E2eGroup { smoke, device, notification, recovery }

E2eGroup currentGroup() {
  const value = String.fromEnvironment('E2E_GROUP', defaultValue: 'smoke');
  return E2eGroup.values.firstWhere(
    (group) => group.name == value,
    orElse: () => E2eGroup.smoke,
  );
}

Future<void> prepareTestScope() async {
  final worker = const String.fromEnvironment('E2E_WORKER', defaultValue: '0');
  await testApi.reset(scope: 'e2e-$worker');
  await mqttFakeOrSandbox.connect(clientId: 'test-$worker');
}
```

이 코드는 테스트마다 같은 계정을 지우는 방식이 아니다. 워커 번호를 Space, MQTT clientId, FCM 테스트 토픽에 모두 반영한다. `device-0`과 `device-1`이 같은 `boiler/command` 토픽을 발행하면 병렬화 이전보다 결과가 더 이상해진다. 테스트 종료 후 삭제하는 것보다 시작 시 해당 워커 범위를 reset하는 편이 CI 중단에도 덜 취약했다.

## GitHub Actions 매트릭스는 작게 시작한다

실제 워커 수를 무작정 늘리면 에뮬레이터 부팅과 APK 설치 시간이 병목이 된다. Android 두 워커부터 측정하고, 테스트 시간이 긴 그룹만 분리했다.

```yaml
strategy:
  fail-fast: false
  matrix:
    group: [smoke, device, notification, recovery]
    worker: [0]

steps:
  - name: Run integration test
    run: >-
      flutter test integration_test/${{ matrix.group }}_test.dart
      -d emulator-5554
      --dart-define=E2E_GROUP=${{ matrix.group }}
      --dart-define=E2E_WORKER=${{ matrix.worker }}
```

내 측정값은 직렬 실행 28분에서 네 그룹 매트릭스 11분으로 줄었다. 그런데 `recovery`만 실패했을 때 전체를 재실행하면 절약한 시간이 사라진다. CI에서는 그룹별 결과 파일과 스크린샷을 보존하고, 실패한 매트릭스만 재시도하도록 구성해야 한다.

## 병렬화하면 안 되는 테스트도 있다

BLE 실기기 한 대, 동일한 Firebase 테스트 계정, 실제 FCM 토큰처럼 독점 자원을 쓰는 테스트는 병렬 그룹에 넣지 않았다. 이 테스트들은 별도 nightly job으로 보내고, 일반 PR에서는 Fake BLE와 테스트 intent를 사용했다.

특히 재시도는 조심해야 한다. 첫 실행이 MQTT 연결을 끊지 못한 상태에서 같은 워커가 재실행되면 이전 clientId의 세션이 남는다. `tearDown`에서 연결 종료를 기다리고, 시작할 때 실행 ID가 붙은 토픽을 정리하는 두 단계를 모두 넣어야 한다.

## 짧게 정리하면

- integration_test 병렬 실행의 핵심은 테스트 파일 분할이 아니라 상태 경계 분리다.
- 워커마다 Space, MQTT clientId, FCM 토픽을 독립적으로 만든다.
- GitHub Actions 매트릭스는 작은 수로 시작하고 그룹별 실행 시간을 측정한다.
- BLE 실기기와 실제 FCM처럼 독점 자원을 쓰는 테스트는 별도 실행으로 분리한다.
- 실패한 그룹만 재시도하려면 로그·스크린샷·결과 파일도 그룹별로 저장해야 한다.

처음에는 CI 머신을 더 크게 사면 해결될 문제라고 생각했다. 실제 병목은 테스트끼리 같은 상태를 만지는 데 있었다. 경계를 분리한 뒤에야 병렬 실행이 속도 향상으로 이어졌다.
