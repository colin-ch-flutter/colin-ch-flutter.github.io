---
layout: post
title: "Flutter integration_test 멀티환경 설정 - IoT 테스트가 운영 MQTT에 붙지 않게 막기"
description: "Flutter integration_test에서 staging과 production 설정이 섞여 운영 MQTT로 연결되는 문제를 테스트 전용 진입점과 환경 가드로 차단하는 방법을 정리했다."
date: 2026-08-13
tags: [Flutter, Dart, integration_test, MQTT, mqtt5_client, IoT, CI/CD]
comments: true
share: true
---

![Flutter integration_test 멀티환경과 IoT CI 실행 가드](/assets/images/flutter-integration-test-parallel-ci-iot.png)

이 그림처럼 테스트 워커마다 실행 환경과 연결 대상을 분리해야 한다. `Flutter integration_test`는 앱을 실제로 띄우므로 설정 하나가 빠지면 production MQTT에 붙을 수 있다.

처음엔 GitHub Actions 명령에 `--dart-define=CONFIG=stage`만 추가하면 충분하다고 생각했다. 그런데 테스트 진입점에서 `const String.fromEnvironment`의 기본값을 `prod`로 둔 탓에, CI YAML 오타가 운영 설정으로 조용히 대체됐다. 테스트는 통과했지만 실제 기기 상태를 건드릴 위험이 있었다.

## 테스트 환경을 별도 값으로 고정하기

운영·스테이징 선택값과 테스트 여부를 하나의 문자열로 처리하지 않고, 테스트 실행 여부를 별도로 둔다.

```dart
class TestRuntime {
  static const isIntegrationTest = bool.fromEnvironment(
    'INTEGRATION_TEST',
    defaultValue: false,
  );

  static const apiBaseUrl = String.fromEnvironment('API_BASE_URL');
  static const mqttHost = String.fromEnvironment('MQTT_HOST');

  static void validate() {
    if (!isIntegrationTest) return;
    if (apiBaseUrl.isEmpty || mqttHost.isEmpty) {
      throw StateError('integration_test 환경값이 비어 있다');
    }
    if (mqttHost.contains('prod')) {
      throw StateError('integration_test에서 production MQTT를 사용할 수 없다');
    }
  }
}
```

`validate()`는 연결 전에 실패시키는 안전장치다. 빈 값일 때 기본값을 넣지 않는 게 핵심이다.

```dart
Future<void> main() async {
  TestRuntime.validate();
  await appMain(environment: AppEnvironment.integrationTest);
}
```

CI 실행 명령도 환경값을 눈에 보이게 적었다.

```bash
flutter test integration_test/boiler_control_test.dart \
  --dart-define=INTEGRATION_TEST=true \
  --dart-define=API_BASE_URL=https://staging-api.example.com \
  --dart-define=MQTT_HOST=staging-mqtt.example.com
```

| 확인 대상 | 위험한 방식 | 테스트에서 적용한 기준 |
|---|---|---|
| 값 누락 | `prod` 기본값 사용 | 누락 즉시 실패 |
| MQTT 연결 | 환경에 따라 자동 추론 | `staging-mqtt`만 허용 |
| 앱 진입점 | 운영 `main()` 재사용 | integrationTest 모드 명시 |
| CI 재실행 | 이전 빌드 설정 재사용 | 명령에 모든 define 기록 |

`dart-define`에 비밀키를 넣으면 안 된다. 빌드 산출물과 CI 로그에 노출될 수 있으므로 endpoint와 모드만 전달하고, 인증 정보는 테스트 전용 Secret과 Fake 서비스로 분리한다.

짧게 정리하면 Flutter integration_test 멀티환경의 핵심은 stage 값을 잘 넣는 데 있지 않다. 값이 빠져도 production으로 흘러가지 않도록 기본값을 없애고, MQTT 연결 직전에 환경 가드를 두는 것이다.
