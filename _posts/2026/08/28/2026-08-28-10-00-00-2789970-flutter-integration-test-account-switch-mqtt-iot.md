---
layout: post
title: "Flutter integration_test 다중 계정 전환 - GetX 상태와 MQTT 구독 격리 검증"
description: "Flutter integration_test로 로그아웃 후 다른 계정에 로그인하는 흐름을 검증하고, GetX 캐시와 MQTT 구독이 이전 스마트홈 사용자에게 남는 문제를 해결하는 방법을 정리했다."
date: 2026-08-28
tags: [Flutter, Dart, integration_test, GetX, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter integration_test 다중 계정 전환과 MQTT 구독 격리](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
*계정 전환 테스트에서 확인할 것은 로그인 화면 이동이 아니라 이전 사용자의 IoT 상태가 정말 사라졌는지다.*

Flutter integration_test에서 로그아웃 뒤 로그인 화면이 보이는지만 확인하면 부족하다. A에서 B로 바꾸자 GetX의 `selectedSpace`와 MQTT `space/A/#` 구독이 남았다.

## 계정 전환에서 끊어야 할 것

JWT만 지우면 메모리 상태와 통신 수명은 남는다. 사용자 경계에서 세 가지를 함께 확인했다.

| 대상 | 로그아웃 전 | 새 로그인 후 |
| --- | --- | --- |
| GetX 상태 | A의 기기·Space | B 기준으로 재생성 |
| MQTT 토픽 | `space/A/#` | `space/B/#`만 활성 |
| 로컬 캐시 | A의 대시보드 | B의 데이터로 재조회 |

로그아웃은 `close → 토큰 삭제 → Controller 제거 → 화면 이동` 순서로 실행한다. `close()`에서 unsubscribe와 listener 종료를 await해야 B 로그인 시점에 A의 소켓 이벤트가 남지 않는다.

## integration_test에서 실제로 검사할 것

화면과 통신 assertion을 함께 둔다. broker 대신 토픽을 기록하는 Fake MQTT를 사용한다.

```dart
testWidgets('계정 전환 시 MQTT 구독을 격리한다', (tester) async {
  final mqtt = RecordingMqtt();
  await launchTestApp(mqtt: mqtt);
  await login(tester, 'a@test.dev');
  expect(mqtt.activeTopics, {'space/A/#'});
  await tester.tap(find.byKey(const Key('logout')));
  await tester.pumpAndSettle();
  expect(mqtt.unsubscriptions, contains('space/A/#'));
  await login(tester, 'b@test.dev');
  expect(find.byKey(const Key('home-screen')), findsOneWidget);
  expect(mqtt.activeTopics, {'space/B/#'});
});
```

처음에는 마지막 `activeTopics` 검증이 없어서 통과했다. 실기기에서만 retained message가 B의 카드에 섞였다. 화면만 검사하면 놓치는 문제다.

로그아웃 때 소켓·Timer·Stream을 닫고 retained message의 `spaceId`도 검증한다. 핵심은 이전 집의 상태가 새 집으로 새지 않는지다.
