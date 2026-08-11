---
layout: post
title: "Flutter integration_test 테스트 계정 인증 - JWT와 SecureStorage 때문에 깨지는 IoT E2E 해결"
description: "Flutter integration_test에서 테스트 계정 로그인과 JWT, flutter_secure_storage가 엉켜 flaky해지는 문제를 테스트 전용 인증 부트스트랩과 세션 초기화로 해결하는 방법을 정리했다."
date: 2026-08-12
tags: [Flutter, Dart, integration_test, JWT, flutter_secure_storage, 인증, IoT, CI/CD]
comments: true
share: true
---

![Flutter integration_test 테스트 계정 인증과 JWT 세션 부트스트랩](https://images.unsplash.com/photo-1563013544-824ae1b704d3?w=1200&q=80)

이 그림에서 봐야 할 부분은 로그인 화면을 억지로 통과시키는 게 아니라, 테스트마다 독립된 인증 세션을 준비하는 흐름이다.

Flutter integration_test에서 테스트 계정 로그인이 가끔 실패한다면 서버보다 `flutter_secure_storage`에 남은 JWT가 원인일 수 있다. 내 경우 첫 테스트는 정상인데 두 번째 테스트부터 다른 Space가 보이거나, 만료된 토큰으로 MQTT 연결이 시작됐다. 처음엔 로그인 버튼을 누르는 시간을 늘렸는데 해결되지 않았다. 해보니 문제는 테스트 순서마다 인증 상태를 지우지 않은 데 있었다.

## 로그인 UI를 매번 거치는 방식의 한계

사람이 사용하는 로그인 플로우는 UI로 검증해야 한다. 하지만 보일러 제어, FCM 딥링크, MQTT 재연결 테스트까지 매번 로그인 UI를 통과시키면 인증이 공통 병목이 된다.

| 방식 | 장점 | 실제로 깨지는 지점 |
|---|---|---|
| 로그인 UI 직접 실행 | 회원가입·로그인 화면까지 검증 | 네트워크, OTP, 키보드 타이밍 |
| 고정 JWT 저장 | 빠르게 홈 진입 | 만료 토큰과 계정 상태 공유 |
| 테스트 API 로그인 | 새 세션을 매번 발급 | 서버 seed와 토큰 전달 필요 |
| 앱 내부 Fake 인증 | 완전히 빠르고 안정적 | 실제 인증 서버 흐름은 검증하지 못함 |

로그인 테스트 하나만 실제 UI를 사용하고, 나머지 IoT E2E는 테스트 API 로그인으로 세션을 준비하는 편이 균형이 좋았다.

## 테스트 전용 인증 부트스트랩

앱 시작 시 `E2E_TEST_MODE`가 켜져 있으면 테스트 계정으로 API 로그인한다. 발급받은 토큰은 일반 로그인과 같은 Repository를 통해 저장해야 테스트 전용 경로가 생기지 않는다.

```dart
Future<void> bootstrapE2eSession() async {
  const email = String.fromEnvironment('E2E_EMAIL');
  const password = String.fromEnvironment('E2E_PASSWORD');

  if (email.isEmpty || password.isEmpty) {
    throw StateError('E2E_EMAIL과 E2E_PASSWORD가 필요하다');
  }

  final authRepository = Get.find<AuthRepository>();
  final result = await authRepository.login(
    email: email,
    password: password,
  );

  await Get.find<SecureSession>().save(
    accessToken: result.accessToken,
    refreshToken: result.refreshToken,
  );
}
```

코드 위의 함수는 화면을 건너뛰기 위한 편법이 아니라 기존 로그인 UseCase와 같은 저장 경로를 쓰기 위한 진입점이다. 테스트 코드에서 `flutter_secure_storage`를 직접 조작하면 Android Keystore와 iOS Keychain 차이가 생긴다. 실행 전에는 디버그용 MethodChannel로 앱의 `SecureSession.clear()`를 호출하고, 테스트 계정의 Space와 기기 데이터를 reset했다.

## 토큰보다 계정 데이터가 더 자주 문제였다

JWT를 새로 발급해도 테스트 계정의 Space와 기기 상태가 남으면 결과는 달라진다. 그래서 인증 초기화와 데이터 초기화를 한 세트로 묶었다. [Flutter integration_test 테스트 데이터 격리]({% post_url 2026-08-10-16-00-00-1864095-flutter-integration-test-data-isolation-iot %})의 seed/reset 패턴을 인증 직후 실행하는 방식이다.

특히 CI에서는 같은 계정을 Android와 iOS 매트릭스가 동시에 사용하지 않게 해야 한다. 계정별로 `E2E_RUN_ID`를 붙여 Space를 만들거나, 적어도 기기 목록과 알림 데이터를 실행별로 분리해야 한다. 고정 계정 하나를 공유하면 한쪽 테스트가 기기를 삭제한 순간 다른 플랫폼 테스트가 연쇄적으로 실패한다.

## 정리

- 로그인 UI는 인증 화면 테스트에 남기고, IoT E2E는 테스트 API 로그인으로 시작한다.
- JWT는 테스트마다 새로 발급하고 `flutter_secure_storage`의 실제 앱 저장 경로를 사용한다.
- 세션 삭제와 테스트 계정 데이터 reset을 앱 실행 전에 함께 수행한다.
- CI 플랫폼 매트릭스에서 테스트 계정을 공유하지 않는다.

처음엔 로그인 성공 화면만 기다리면 충분하다고 생각했다. 실제로는 토큰 저장, Space seed, MQTT 연결 준비를 분리해서 기다려야 했다. 인증 부트스트랩을 고정하면 BLE와 MQTT 시나리오가 실패했을 때 원인을 기기 제어 쪽으로 좁힐 수 있다.
