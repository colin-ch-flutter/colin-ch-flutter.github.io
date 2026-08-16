---
layout: post
title: "Flutter integration_test 앱 초기화 순서 - IoT MQTT 연결 전 화면 진입 막기"
description: "Flutter integration_test에서 IoT 앱 시작 시 인증 복원·MQTT 초기화·첫 화면 진입이 엇갈리는 문제를 재현하고, 앱 부트스트랩 완료 신호와 테스트 전용 appRunner로 초기화 순서를 고정하는 방법을 정리했다."
date: 2026-08-17
tags: [Flutter, Dart, integration_test, MQTT, mqtt5_client, IoT, 스마트홈, Riverpod, CI/CD]
comments: true
share: true
---

![Flutter integration_test IoT 앱 초기화 순서와 MQTT 연결 테스트](/assets/images/flutter-integration-test-ci-report-iot.png)

이 그림에서 볼 부분은 앱을 띄웠다는 사실보다 인증 복원, MQTT 연결, 첫 기기 상태 수신이 정해진 순서로 끝났는지 확인하는 것이다.

Flutter integration_test에서 앱이 뜨고 `find.text('거실')`만 찾으면 준비가 끝난 줄 알았다. 그런데 CI에서는 화면이 먼저 열리고 MQTT 연결이 늦게 붙으면서 보일러 카드가 “알 수 없음”으로 잠깐 표시됐다. 테스트는 버튼을 눌렀지만 연결 전 명령이라 버려졌고, 로컬에서는 네트워크가 빨라서 계속 통과했다.

## 앱 시작 시 섞이면 안 되는 세 단계

IoT 앱의 초기화는 보통 세 작업으로 나뉜다. 문제는 각각의 Future를 `unawaited`로 흘려보내고 `runApp()`부터 호출할 때 생긴다.

| 단계 | 완료 기준 | 완료 전에 화면을 열면 생기는 문제 |
|---|---|---|
| 인증 복원 | SecureStorage에서 토큰과 사용자 공간을 읽음 | 로그인 화면이 잠깐 보였다가 대시보드로 튐 |
| MQTT 준비 | 연결 후 필요한 토픽을 구독함 | 첫 명령이 연결 전 publish 됨 |
| 상태 동기화 | 기기별 최신 상태를 한 번 이상 수신함 | 오래된 캐시를 최신 상태로 오인함 |

처음에는 `Future.delayed(const Duration(seconds: 2))`를 테스트에 넣었다. 해보니 기기마다 필요한 시간이 달랐고, 2초를 기다려도 연결 완료를 보장하지 못했다. 시간 대신 앱이 정말 준비됐다는 상태를 노출하는 편이 맞았다.

## 부트스트랩 완료 신호 만들기

화면이 임의의 Provider를 여러 개 기다리지 않도록 앱 시작 상태를 하나로 묶었다. 아래 상태가 `ready`가 된 뒤에만 대시보드 라우트를 열도록 했다.

```dart
enum BootstrapStatus { loading, ready, failure }

class AppBootstrap {
  AppBootstrap({
    required this.auth,
    required this.mqtt,
    required this.deviceState,
  });

  final AuthRepository auth;
  final MqttService mqtt;
  final DeviceStateRepository deviceState;

  BootstrapStatus status = BootstrapStatus.loading;

  Future<void> run() async {
    try {
      final session = await auth.restoreSession();
      if (session == null) {
        status = BootstrapStatus.ready;
        return;
      }

      await mqtt.connect(session.accessToken);
      await mqtt.subscribeDeviceState(session.spaceId);
      await deviceState.waitForInitialSnapshot();
      status = BootstrapStatus.ready;
    } catch (_) {
      status = BootstrapStatus.failure;
      rethrow;
    }
  }
}
```

여기서 `waitForInitialSnapshot()`은 영원히 기다리면 안 된다. 실제 구현에서는 5초 타임아웃을 두고, 캐시를 보여줄지 오류 화면으로 보낼지 정책을 정했다. 핵심은 MQTT `connected`만 보고 준비 완료로 처리하지 않는 것이다. 연결됐어도 구독 메시지가 아직 오지 않았다면 기기 상태는 준비되지 않았다.

## integration_test용 appRunner

테스트는 화면이 나타난 뒤 상태를 추측하지 않고 부트스트랩 완료를 기다린다. 외부 브로커를 사용하지 않는 CI에서는 `FakeMqttService`가 구독 완료와 초기 snapshot을 명시적으로 발생시킨다.

```dart
Future<void> appRunner(
  WidgetTester tester, {
  required FakeMqttService mqtt,
}) async {
  final bootstrap = AppBootstrap(
    auth: FakeAuthRepository.loggedIn(spaceId: 'space-test'),
    mqtt: mqtt,
    deviceState: FakeDeviceStateRepository(mqtt),
  );

  await tester.pumpWidget(TestApp(bootstrap: bootstrap));
  await tester.pump();

  await tester.runAsync(bootstrap.run);
  await tester.pumpAndSettle();

  expect(find.byKey(const Key('dashboard-ready')), findsOneWidget);
}

testWidgets('앱 시작 후 MQTT 초기 상태를 받은 뒤 제어한다', (tester) async {
  final mqtt = FakeMqttService(autoPublishSnapshot: true);

  await appRunner(tester, mqtt: mqtt);

  expect(find.text('거실 보일러 꺼짐'), findsOneWidget);
  expect(mqtt.subscribeCount, 1);

  await tester.tap(find.byKey(const Key('boiler-power')));
  expect(mqtt.publishedCommands, ['heater_on']);
});
```

`pumpAndSettle()`은 부트스트랩을 대신하지 않는다. 앱 내부에 계속 살아 있는 MQTT listener나 애니메이션이 있으면 settle되지 않을 수도 있다. 테스트가 기다려야 하는 것은 프레임이 조용해지는 순간이 아니라 `dashboard-ready`라는 명시적인 준비 상태다.

## 실제로 잡힌 초기화 경쟁

한 번은 토큰 복원이 늦어지면서 익명 clientId로 MQTT 연결이 먼저 생성됐다. 이후 인증된 clientId로 다시 연결됐지만, 첫 연결의 listener가 남아 기기 상태가 두 번씩 들어왔다. `run()`을 한 번만 호출하도록 보호하고, 실패한 bootstrap에서 만든 MQTT client를 `dispose()`하는 정리까지 넣었다.

| 실패 로그 | 원인 | 수정 기준 |
|---|---|---|
| `publish before connected` | 화면 진입이 MQTT 연결보다 빠름 | `bootstrap.ready` 이후 버튼 활성화 |
| `subscribeCount == 2` | 재시도 때 이전 client 미정리 | 실패 경로에서 `dispose` 호출 |
| 오래된 상태 표시 | 연결만 완료하고 snapshot 미대기 | 첫 상태 수신을 준비 조건에 포함 |

핵심은 `runApp()`을 늦추는 것이 아니다. 앱은 빨리 띄우되, 사용자 제어 화면과 MQTT 명령 버튼을 부트스트랩 상태 뒤에 두는 것이다. Flutter integration_test에서도 `Future.delayed` 대신 인증·연결·초기 상태 수신을 하나의 완료 신호로 묶으면 기기 속도와 CI 네트워크에 덜 흔들리는 IoT 테스트가 된다.
