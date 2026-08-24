---
layout: post
title: "Flutter integration_test MQTT 구독 복구 검증 - 재연결 후 IoT 이벤트 누락 막기"
description: "Flutter integration_test에서 MQTT 연결이 복구된 뒤 구독 토픽을 다시 등록하는지 검증하고, 스마트홈 화면의 늦은 기기 이벤트 누락을 막는 방법을 정리했다."
date: 2026-08-24
tags: [Flutter, Dart, IoT, MQTT, mqtt5_client, 스마트홈, CI/CD]
comments: true
share: true
---

![Flutter integration_test MQTT 구독 복구와 스마트홈 기기 이벤트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

이 그림에서 볼 부분은 연결 아이콘이 아니라, 재연결 뒤에도 각 기기의 상태 이벤트가 계속 들어오는지다.

Flutter integration_test에서 MQTT `connect()` 성공만 확인하면 실제 장애를 놓친다. 연결은 복구됐지만 기존 구독이 사라져 보일러 상태 이벤트를 받지 못하는 경우가 있기 때문이다. 처음엔 연결 상태만 갱신했다가, 네트워크 복구 후 새 이벤트가 멈추는 버그를 만났다.

## 연결 복구와 구독 복구는 별개다

MQTT 클라이언트의 재연결 콜백이 호출됐다는 사실은 토픽 구독까지 끝났다는 뜻이 아니다. Space별 토픽을 동적으로 만들기 때문에, 연결이 다시 된 뒤 현재 Space의 토픽을 재등록해야 했다.

| 시나리오 | 확인할 상태 | 놓치기 쉬운 버그 |
|---|---|---|
| 최초 연결 | 연결 후 구독 등록 | 첫 화면에서 이벤트 수신 지연 |
| 일시적인 연결 끊김 | 재연결 콜백 호출 | 연결됨인데 구독이 없음 |
| Space 변경 후 재연결 | 현재 토픽만 구독 | 이전 Space 이벤트가 섞임 |

구독 책임을 연결 서비스 안에 두고, 복구 시 같은 메서드를 호출하도록 만들었다. 구독 목록은 한 곳에서 관리해야 중복 등록을 추적하기 쉽다.

```dart
Future<void> restoreSubscriptions() async {
  final id = currentSpaceId;
  if (id == null || !mqtt.isConnected) return;

  for (final suffix in ['state', 'event']) {
    await mqtt.subscribe('space/$id/device/+/$suffix');
  }
}
```

## integration_test에서는 끊김을 직접 주입한다

실제 브로커와 Wi-Fi를 테스트에 넣으면 CI가 불안정해진다. 테스트 전용 클라이언트에 연결 끊김과 복구를 주입하고, 구독 호출 기록을 검증했다.

```dart
testWidgets('MQTT 재연결 뒤 현재 Space 토픽을 다시 구독한다', (tester) async {
  final mqtt = FakeMqttClient();
  final session = MqttSession(mqtt)..spaceId = 'home-1';

  await session.onConnected();
  mqtt.disconnectForTest();
  mqtt.reconnectForTest();
  await session.onConnected();

  expect(mqtt.subscribeCalls.where((t) => t.endsWith('/state')).length, 2);
  expect(mqtt.subscribeCalls.where((t) => t.endsWith('/event')).length, 2);
});
```

`pumpAndSettle()`만 호출하면 안 된다. `session.onConnected()`가 끝난 뒤 이벤트를 주입하고, 화면 텍스트가 바뀌는지와 구독 호출 횟수를 함께 확인해야 한다.

## 실제로 막아야 했던 실수

재연결할 때마다 구독을 무조건 추가하면 같은 이벤트가 두 번 들어올 수 있다. 현재 Space를 읽기 전에 복구 콜백이 실행돼 이전 Space 토픽을 다시 구독하는 문제도 있었다. 연결 복구 순서를 `현재 Space 확정 → 연결 확인 → 구독 등록 → 이벤트 수신`으로 고정했다.

Flutter integration_test의 MQTT 검증 기준은 `connected == true`가 아니다. 재연결 뒤 현재 Space의 구독이 복구되고 기기 이벤트가 화면까지 전달되는지를 함께 확인해야 “연결됐는데 상태가 멈춘” 버그를 CI에서 잡을 수 있다.
