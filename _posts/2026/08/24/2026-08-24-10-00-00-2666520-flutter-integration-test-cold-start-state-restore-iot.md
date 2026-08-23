---
layout: post
title: "Flutter integration_test 콜드 스타트 검증 - IoT 초기 상태 유실 막기"
description: "Flutter integration_test에서 앱을 완전히 종료한 뒤 다시 실행하는 콜드 스타트와 백그라운드 복귀를 구분하고, MQTT·Realm 초기 상태가 유실되지 않는지 검증하는 방법을 정리했다."
date: 2026-08-24
tags: [Flutter, Dart, IoT, MQTT, mqtt5_client, 스마트홈, CI/CD]
comments: true
share: true
---

![Flutter integration_test 콜드 스타트와 IoT 상태 복원](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

이 그림에서 볼 부분은 앱을 다시 열었다는 사실이 아니라, 마지막으로 선택한 공간과 기기 상태가 같은 순서로 복원되는지다.

Flutter integration_test에서 `resume`만 검증하면 앱 재시작 버그를 놓친다. 스마트홈 앱은 백그라운드에서 돌아온 경우와 프로세스가 죽었다가 다시 뜬 경우의 초기화 경로가 다르기 때문이다. 내 경우에는 화면은 정상적으로 열렸지만 Realm 캐시를 읽기 전에 MQTT 구독부터 시작해, 보일러 카드가 잠깐 기본값으로 표시됐다.

## 콜드 스타트와 웜 스타트는 다른 시나리오다

처음에는 두 테스트에서 같은 `pumpAndSettle()`을 호출하면 충분하다고 생각했다. 실제로는 로컬 저장소와 구독 순서가 핵심이었다.

| 실행 상태 | 검증할 것 | 흔한 실패 |
|---|---|---|
| 콜드 스타트 | 토큰·Space·Realm 복원 후 MQTT 연결 | 기본 Space로 잠깐 진입 |
| 웜 스타트 | 기존 Controller와 구독 재사용 여부 | 같은 토픽을 두 번 구독 |
| 강제 종료 후 재실행 | 저장된 기기 상태와 서버 동기화 | 이전 화면 상태가 사라짐 |

테스트 앱에서는 초기화 완료를 화면 프레임 수가 아니라 명시적인 Future로 알리게 했다. 이 신호가 있어야 테스트가 MQTT 연결보다 먼저 화면을 찾지 않는다.

```dart
class AppBootstrap {
  final Future<void> ready;

  AppBootstrap({required this.ready});
}

Future<void> startTestApp() async {
  final bootstrap = AppBootstrap(
    ready: restoreSessionAndLocalState(),
  );
  await bootstrap.ready;
  runApp(const SmartHomeApp());
}
```

핵심은 `runApp`을 몇 초 늦추는 게 아니라, 부트스트랩 완료 Future를 GetX나 Riverpod 경계에서 기다릴 수 있게 만드는 것이다. 고정 지연은 기기 성능에 따라 다시 흔들린다.

## 테스트에서는 상태를 만들고, 복원을 확인한다

테스트 전에는 Fake 저장소에 마지막 Space와 보일러 상태를 넣었다. 재실행 후 첫 화면의 카드와 MQTT 구독 횟수를 함께 확인했다.

```dart
testWidgets('cold start restores selected space before mqtt subscribe',
    (tester) async {
  fakeStorage.seedSpace('home-1');
  fakeRealm.seedBoiler('boiler-1', isOn: true);

  await launchTestApp(tester);
  await tester.pumpUntilFound(find.text('거실 보일러'));

  expect(find.text('켜짐'), findsOneWidget);
  expect(fakeMqtt.subscribeCallCount, 1);
});
```

실제 MQTT 브로커를 붙이지 않은 이유는 인터넷 연결이 아니라 복원 완료와 구독 시작의 순서를 검증하기 위해서다. `FakeMqttService`는 구독 횟수만 기록하고 서버 동기화는 별도 환경에서 확인했다.

## 짧게 정리하면

`resumed` 이벤트 테스트가 통과해도 콜드 스타트가 안전하다는 뜻은 아니다. 앱 종료 후 재실행, 저장 상태 복원, MQTT 구독 시작을 각각 관찰 가능한 신호로 만들고 순서를 검증해야 한다. 특히 초기 화면이 보였다는 사실보다 **올바른 Space와 최신 기기 상태가 보였는지**를 assertion으로 남겨야 CI에서만 발생하는 초기화 경합을 잡을 수 있다.
