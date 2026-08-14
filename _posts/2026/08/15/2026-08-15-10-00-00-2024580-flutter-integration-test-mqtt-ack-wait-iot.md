---
layout: post
title: "Flutter integration_test MQTT ACK 대기 - IoT 제어 테스트 flaky 해결"
description: "Flutter integration_test에서 MQTT 명령 ACK를 pumpAndSettle()로 기다리면 간헐적으로 실패한다. IoT 보일러 제어 상태를 명시적으로 기다리고 타임아웃을 잡는 방법을 실제 코드로 정리했다."
date: 2026-08-15
tags: [Flutter, Dart, integration_test, MQTT, mqtt5_client, IoT, 스마트홈, 테스트]
comments: true
share: true
---

![Flutter integration_test MQTT ACK 대기](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Flutter integration_test에서 MQTT ACK를 기다릴 때 `pumpAndSettle()`만 호출하면 안 된다. Flutter 프레임이 멈추는 시점과 MQTT 메시지가 도착하는 시점은 서로 다르기 때문이다. IoT 보일러 제어 테스트에서는 화면의 로딩이 끝났는데도 ACK가 아직 오지 않아, 전체 테스트가 가끔 10초 타임아웃으로 죽었다.

## 문제가 생긴 코드

버튼을 누른 뒤 화면이 안정될 때까지 기다리면 충분하다고 생각했다.

```dart
await tester.tap(find.text('난방 켜기'));
await tester.pumpAndSettle();

expect(find.text('난방 중'), findsOneWidget);
```

이 코드는 로컬 상태만 바뀌는 화면에서는 잘 동작한다. 하지만 실제 흐름은 `publish → 브로커 전달 → 기기 처리 → ACK 수신 → Controller 상태 변경 → 화면 갱신`이다. `pumpAndSettle()`은 이 네트워크 흐름을 기다려주지 않는다.

| 대기 대상 | `pumpAndSettle()`의 역할 | 실제 필요한 처리 |
|---|---|---|
| 애니메이션·프레임 | 프레임이 안정될 때까지 펌프 | 가능 |
| MQTT ACK | 알 수 없음 | 상태 조건을 직접 대기 |
| 연결 끊김 | 알 수 없음 | timeout과 오류 상태 확인 |
| 늦게 도착한 메시지 | 테스트 종료 후 도착 가능 | 구독 해제·정리 |

## 상태를 기다리는 헬퍼

테스트에서 MQTT 클라이언트를 직접 기다리지 않고, 사용자에게 보이는 Controller 상태를 기준으로 대기하도록 만들었다. `pump()`를 짧게 반복하되 무한 대기는 막았다.

```dart
Future<void> pumpUntil(
  WidgetTester tester,
  bool Function() condition, {
  Duration timeout = const Duration(seconds: 5),
}) async {
  final end = DateTime.now().add(timeout);

  while (!condition()) {
    if (DateTime.now().isAfter(end)) {
      throw TimeoutException('상태 대기 시간 초과');
    }
    await tester.pump(const Duration(milliseconds: 100));
  }
}
```

코드 바로 아래에서 실제 보일러 제어 테스트에 적용한다.

```dart
await tester.tap(find.text('난방 켜기'));
await tester.pump();

await pumpUntil(
  tester,
  () => controller.commandState.value == CommandState.acknowledged,
);

expect(find.text('난방 중'), findsOneWidget);
```

처음에는 `Duration(seconds: 30)`으로 잡았다. 실패는 줄었지만 CI에서 오래 멈춘 뒤 실패해서 원인을 찾기 더 어려웠다. 실제 기기 응답이 보통 1초 안에 오는 환경이라 timeout은 5초로 두고, 테스트 로그에 마지막 상태와 commandId를 남기는 편이 낫다.

## ACK가 다른 명령에 섞이지 않게 하기

MQTT 토픽만 확인하면 이전 테스트의 ACK를 현재 명령의 응답으로 오인할 수 있다. 발행한 명령의 `commandId`와 ACK의 값을 함께 비교해야 한다.

```dart
final commandId = controller.sendHeatingCommand(enabled: true);

await pumpUntil(
  tester,
  () => controller.lastAck.value?.commandId == commandId &&
      controller.lastAck.value?.status == 'accepted',
);
```

실제 코드에서는 `sendHeatingCommand`가 `Future<String>`을 반환하도록 바꿨다.

```dart
final commandId = await controller.sendHeatingCommand(enabled: true);
```

이렇게 해야 테스트가 “난방 켜기 ACK가 왔다”를 검증하지, “어떤 명령인지 모르는 accepted가 왔다”를 검증하지 않는다.

## 실패 로그와 정리

timeout이 났을 때 `commandId`, 현재 연결 상태, 마지막 ACK를 출력하면 기기 문제와 테스트 문제를 구분하기 쉽다. 테스트가 끝나면 MQTT 구독과 Fake 브로커의 StreamController도 닫아야 한다. 정리를 빼먹으면 다음 테스트가 이전 ACK를 받아서 순서에 따라 결과가 달라진다.

```dart
addTearDown(() async {
  await fakeMqtt.disconnect();
  controller.dispose();
});
```

실기기 integration_test라면 테스트마다 새 `commandId`를 만들고, 테스트 전용 토픽과 계정을 사용해야 한다. 운영 토픽에 연결한 채 timeout만 늘리는 건 실패를 숨기는 방법에 가깝다.

## 짧게 정리하면

Flutter integration_test에서 MQTT ACK는 `pumpAndSettle()`의 대기 대상이 아니다. 화면 프레임이 아니라 Controller의 ACK 상태와 `commandId`를 조건으로 기다려야 한다. timeout은 실제 응답 시간보다 조금 넉넉하게 잡고, 실패 로그와 MQTT 정리까지 넣어야 테스트 순서에 따른 간헐 실패를 줄일 수 있다.
