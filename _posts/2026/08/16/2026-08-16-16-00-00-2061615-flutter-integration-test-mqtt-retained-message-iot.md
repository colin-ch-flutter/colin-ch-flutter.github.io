---
layout: post
title: "Flutter integration_test MQTT retained message 테스트 - 재진입 중복 반영 막기"
description: "Flutter integration_test에서 MQTT retained message가 앱 재진입 때 중복 반영되는 문제를 재현하고, 구독 초기화·메시지 식별자·상태 검증으로 IoT 제어 화면을 안정화하는 방법을 정리했다."
date: 2026-08-16
tags: [Flutter, Dart, integration_test, MQTT, mqtt5_client, IoT, 스마트홈]
comments: true
share: true
---

![Flutter integration_test MQTT retained message 중복 반영 테스트](/assets/images/flutter-integration-test-ci-report-iot.png)

이 그림에서 볼 부분은 앱 재진입 뒤 같은 MQTT 상태가 한 번만 반영되는지다.

Flutter integration_test에서 MQTT 연결 복구만 확인하면 충분한 줄 알았다. 그런데 스마트홈 앱을 다시 열자 보일러 상태 카드의 이벤트 로그가 두 번씩 쌓였다. 브로커 중복 발행이 아니라 `retained` 상태를 새 이벤트처럼 추가한 것이 원인이었다.

## 문제 상황 — retained message는 새 메시지처럼 도착한다

MQTT retained message는 토픽의 마지막 값을 저장해 둔다. 새로 구독하면 기기가 다시 발행하지 않아도 브로커가 마지막 값을 즉시 전달한다.

| 앱 상태 | 발생한 동작 | 화면 결과 |
| --- | --- | --- |
| 최초 실행 | 토픽 구독 후 retained 수신 | 상태 카드 1회 갱신 |
| 백그라운드 복귀 | 구독 객체 재생성 | 같은 상태가 2회 반영 |
| 테스트 재실행 | 로컬 이벤트 목록 미초기화 | 이전 실행 결과까지 섞임 |

`mqtt5_client`의 `connected` 상태만으로는 구독이 한 번뿐인지 알 수 없다. 연결 수, 구독 수, 메시지 반영 횟수를 따로 확인해야 했다.

## 구독과 상태 반영을 분리한다

MQTT callback에서 바로 이벤트를 추가하지 말고 메시지 식별자를 검사한다. 테스트에는 retained 응답을 두 번 보내는 Fake 서비스를 주입했다.

```dart
final handled = <String>{};
void apply(MqttMessage message) {
  final id = message.properties['messageId'];
  if (id != null && !handled.add(id)) return;
  events.add(DeviceEvent.fromMqtt(message));
}
```

장치가 `messageId`를 보내지 않으면 `topic + reportedAt + payload hash`처럼 앱에서 키를 만든다. payload만 비교하면 같은 온도의 정상 보고까지 버릴 수 있다.

## integration_test에서 재현하기

앱을 열고 백그라운드 복귀를 실행한 뒤, 화면 상태와 구독·이벤트 반영 횟수를 함께 검증한다.

```dart
testWidgets('retained 상태를 중복 반영하지 않는다', (tester) async {
  final fakeMqtt = FakeMqttService(retainedMessage: boilerMessage);
  await launchTestApp(tester, mqtt: fakeMqtt);

  await tester.binding.handleAppLifecycleStateChanged(AppLifecycleState.paused);
  await tester.binding.handleAppLifecycleStateChanged(AppLifecycleState.resumed);
  expect(fakeMqtt.subscribeCount, 1);
  expect(find.text('현재 온도 21℃'), findsOneWidget);
  expect(find.byKey(const Key('device-event')), findsOneWidget);
}
```

`subscribeCount`가 2면 기존 subscription을 취소하지 않은 것이다. 구독은 한 번인데 이벤트가 두 개면 deduplication 키가 없거나, retained 메시지를 별도 상태 갱신으로 분리하지 못한 경우다.

앱이 `resumed`가 될 때마다 `subscribe()`를 호출한 것이 첫 번째 실수였다. 연결이 살아 있으면 재구독하지 않고, 끊겼을 때만 `unsubscribe → connect → subscribe` 순서를 탄다. Fake stream과 이벤트 저장소는 `tearDown`에서 초기화한다.

짧게 정리하면 `Flutter integration_test`의 MQTT retained message 테스트는 “연결됐는가” 하나로 끝나지 않는다.

- lifecycle 재진입 때 MQTT 구독이 한 번만 생성되는가
- retained message와 실제 변경 이벤트를 구분하는가
- 같은 메시지 식별자가 화면 이벤트에 두 번 반영되지 않는가
- 테스트 종료 후 Fake stream과 상태가 초기화되는가

이 네 가지를 함께 확인해야 앱을 다시 열 때 상태는 복원하면서 이벤트 중복은 막을 수 있다.
