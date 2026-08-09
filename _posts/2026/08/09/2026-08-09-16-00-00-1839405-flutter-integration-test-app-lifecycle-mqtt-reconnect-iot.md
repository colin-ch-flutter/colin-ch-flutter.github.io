---
layout: post
title: "Flutter integration_test 앱 생명주기 - 백그라운드 복귀 후 MQTT 재연결 검증"
description: "Flutter integration_test로 IoT 앱이 백그라운드에서 복귀할 때 MQTT 재연결과 기기 상태 동기화를 검증하는 방법을 정리했다. 고정 delay 대신 앱 생명주기와 연결 상태를 기다리는 테스트 코드도 함께 다룬다."
date: 2026-08-09
tags: [Flutter, integration_test, MQTT, mqtt5_client, IoT, Android, iOS]
comments: true
share: true
---

![Flutter integration_test 앱 생명주기와 MQTT 재연결](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

이 그림에서 봐야 할 부분은 앱 화면이 다시 보이는 순간과 MQTT 연결이 다시 살아나는 순간이 같지 않다는 점이다.

`Flutter integration_test`에서 버튼 제어만 검증하면 백그라운드 복귀 버그를 놓치기 쉽다. 스마트홈 앱은 사용자가 알림을 확인하거나 다른 앱을 잠깐 다녀온 뒤 돌아오는 일이 많다. 처음엔 `AppLifecycleState.resumed`가 오면 곧바로 MQTT를 재연결하면 된다고 생각했다. 해보니 화면은 resumed인데 소켓은 아직 끊긴 상태라, 복귀 직후 보일러 명령이 사라지는 경우가 있었다.

## 생명주기와 연결 상태는 따로 움직인다

| 단계 | 앱에서 보이는 상태 | 테스트가 확인할 것 |
|---|---|---|
| 백그라운드 진입 | `paused` 또는 `inactive` | 기존 구독과 타이머 정리 |
| 포그라운드 복귀 | `resumed` | 재연결 요청이 한 번만 발생 |
| 브로커 연결 완료 | connected | 토픽 재구독 및 최신 상태 반영 |
| 동기화 완료 | device state updated | 화면에 오래된 값이 남지 않음 |

`resumed` 이벤트만 기다리면 마지막 행까지 검증하지 못한다. 테스트에서는 앱 생명주기 이벤트를 발생시킨 뒤 Fake MQTT 서비스가 `connected`와 `stateSynced`를 차례로 내보내도록 만들었다. 실제 브로커를 붙이지 않아도 복귀 로직의 순서를 확인할 수 있다.

```dart
testWidgets('백그라운드 복귀 후 MQTT를 재연결하고 상태를 동기화한다',
    (tester) async {
  final mqtt = FakeMqttService();
  await tester.pumpWidget(TestApp(mqtt: mqtt));

  tester.binding.handleAppLifecycleStateChanged(
    AppLifecycleState.paused,
  );
  tester.binding.handleAppLifecycleStateChanged(
    AppLifecycleState.resumed,
  );
  await tester.pump();

  expect(mqtt.reconnectCalls, 1);
  mqtt.emitConnected();
  mqtt.emitStateSynced({'boiler': 'on'});
  await tester.pump();

  expect(find.text('켜짐'), findsOneWidget);
});
```

코드 바로 위에서 생명주기 이벤트를 직접 넣은 이유는 기기 화면을 실제로 껐다 켜는 동작과 앱 내부 상태 전환을 분리하기 위해서다. `FakeMqttService`에는 `reconnectCalls` 카운터를 두고, 연결 완료 전에는 상태 동기화가 실행되지 않도록 했다. 이 검사가 없으면 재연결 버튼을 두 번 누른 것처럼 중복 연결이 생겨 실제 기기에서 구독 콜백도 중복될 수 있다.

## 실제 실기기 테스트에서 남길 경계

Fake 테스트는 순서와 상태 전이를 보장하지만 Android·iOS가 백그라운드에서 소켓을 유지하는 방식까지 보장하지 않는다. 그래서 범위를 나눴다.

| 검증 대상 | Fake integration_test | Android·iOS 실기기 |
|---|---:|---:|
| `paused → resumed` 처리 | O | O |
| 재연결 호출 횟수 | O | O |
| 실제 네트워크 단절 복구 | X | O |
| MQTT 브로커 세션 만료 | X | 별도 smoke |

주의할 점은 테스트가 끝날 때 MQTT 스트림과 재연결 타이머를 반드시 닫는 것이다. 처음에는 `tearDown`에서 앱만 dispose했는데, 다음 테스트가 시작되자 이전 타이머가 다시 연결을 호출했다. 결과는 테스트마다 달랐고, CI에서만 간헐적으로 실패했다.

짧게 정리하면 `resumed`는 재연결의 시작 신호일 뿐이다. Flutter IoT 앱의 `integration_test`는 생명주기 이벤트, MQTT 연결 완료, 기기 상태 동기화를 각각 기다리고 검증해야 백그라운드 복귀 버그를 잡을 수 있다.
