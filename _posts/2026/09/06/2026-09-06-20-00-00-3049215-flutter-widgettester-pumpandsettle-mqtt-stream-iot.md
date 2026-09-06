---
layout: post
title: "Flutter WidgetTester pumpAndSettle 무한 대기 - MQTT 상태 스트림 테스트 타임아웃 해결"
description: "Flutter WidgetTester에서 MQTT 상태 스트림과 로딩 애니메이션 때문에 pumpAndSettle이 끝나지 않는 문제를 재현하고, pump(Duration)와 제한 시간 설정으로 IoT 화면 테스트를 안정화하는 방법을 정리했다."
date: 2026-09-06
tags: [Flutter, Dart, 테스트, WidgetTester, MQTT, IoT, 스마트홈]
comments: true
share: true
---
![Flutter WidgetTester로 MQTT 상태 스트림 화면 테스트](https://images.unsplash.com/photo-1558346490-a72e53ae2d4f?w=1200&q=80)

Flutter WidgetTester에서 `pumpAndSettle()`을 호출했는데 테스트가 10분 뒤 타임아웃으로 실패했다. 처음엔 테스트 코드의 비동기 처리가 잘못된 줄 알았는데, 원인은 MQTT 상태 스트림과 계속 살아 있는 애니메이션이었다. IoT 화면에서는 “모든 프레임이 멈출 때까지” 기다리는 방식보다 필요한 상태 변화만 `pump()`으로 진행하는 편이 안전하다.

## `pumpAndSettle()`이 끝나지 않는 화면

보일러 카드가 MQTT 수신 스트림을 구독하고, 연결 중에는 `CircularProgressIndicator`를 보여준다고 하자. 아래 테스트는 얼핏 자연스럽다.

```dart
await tester.tap(find.byKey(const Key('connect-button')));
await tester.pumpAndSettle();

expect(find.text('연결됨'), findsOneWidget);
```

하지만 연결 상태 스트림이 주기적으로 heartbeat 이벤트를 보내거나 로딩 위젯의 애니메이션이 계속 동작하면 Flutter에는 아직 처리할 프레임이 남아 있다. `pumpAndSettle()`은 애니메이션이 끝날 때까지 반복해서 `pump()`하므로, 끝나지 않는 화면에서는 테스트도 끝나지 않는다.

| 상황 | 적합한 호출 | 이유 |
|---|---|---|
| 한 번 실행되는 페이지 전환 | `pumpAndSettle()` | 모든 애니메이션 종료를 확인 |
| MQTT·BLE 스트림이 계속 동작 | `pump(Duration)` | 필요한 시간만 진행 |
| 끝나야 하는 애니메이션 | `pumpAndSettle(timeout: ...)` | 무한 대기를 조기에 발견 |
| 버튼 탭 직후 상태 확인 | `pump()` | 한 프레임만 갱신 |

## Fake 스트림의 이벤트를 직접 주입한다

실제 MQTT 연결을 기다리지 않고, 테스트가 상태 이벤트를 원하는 순간에 발행하도록 Fake를 만든다.

```dart
class FakeMqttService implements MqttService {
  final statusController = StreamController<MqttStatus>.broadcast();

  @override
  Stream<MqttStatus> get statusStream => statusController.stream;

  void emit(MqttStatus status) => statusController.add(status);

  Future<void> dispose() => statusController.close();
}
```

연결 버튼을 누른 뒤 `connected` 이벤트를 넣고, 한 프레임만 진행하면 된다.

```dart
testWidgets('MQTT 연결 이벤트 뒤 보일러 카드가 연결 상태가 된다',
    (tester) async {
  final mqtt = FakeMqttService();
  addTearDown(mqtt.dispose);

  await tester.pumpWidget(TestApp(mqttService: mqtt));
  await tester.tap(find.byKey(const Key('connect-button')));
  await tester.pump();

  expect(find.text('연결 중'), findsOneWidget);

  mqtt.emit(MqttStatus.connected);
  await tester.pump();

  expect(find.text('연결됨'), findsOneWidget);
  expect(find.byType(CircularProgressIndicator), findsNothing);
});
```

여기서 중요한 건 `emit()`과 `pump()`을 한 쌍으로 보는 것이다. 스트림에 이벤트를 넣었다고 위젯 트리가 즉시 바뀌지는 않는다. 이벤트 큐와 프레임을 처리할 기회를 줘야 화면의 `Obx`, `StreamBuilder` 또는 상태 관리 위젯이 다시 빌드된다.

## 애니메이션만 필요한 시간만 흘린다

연결 성공 후 300ms 동안 토스트가 표시되는 기능이라면 전체 settle을 기다리지 않고 정확히 그 시간만 진행한다.

```dart
mqtt.emit(MqttStatus.connected);
await tester.pump();
await tester.pump(const Duration(milliseconds: 300));

expect(find.text('기기 연결 완료'), findsOneWidget);
```

반대로 정말 끝나야 하는 애니메이션에는 제한 시간을 둔다. 제한 시간이 없으면 회귀 원인을 찾기 전에 긴 기본 타임아웃을 기다리게 된다.

```dart
await tester.pumpAndSettle(
  duration: const Duration(milliseconds: 50),
  phase: EnginePhase.sendSemanticsUpdate,
  timeout: const Duration(seconds: 2),
);
```

다만 이 호출을 MQTT 스트림 자체가 멈추기를 기대하는 테스트에 사용하면 안 된다. 연결 스트림의 생명주기는 화면 애니메이션과 다르다. 스트림은 테스트 종료 시 `addTearDown()`으로 닫고, 화면 상태 검증은 이벤트 단위로 끝내는 게 책임이 분명하다.

## 실제로 막혔던 지점

처음에는 모든 Widget Test 끝에 `pumpAndSettle()`을 넣었다. 로컬에서는 통과했지만 CI에서 MQTT mock의 heartbeat 타이머가 살아 있는 테스트만 간헐적으로 멈췄다. `pump()`으로 바꾸자 빨라졌고, Fake의 `StreamController`를 닫지 않아 다음 테스트까지 이벤트가 남던 문제도 `addTearDown()`으로 같이 해결됐다.

정리하면 기준은 단순하다.

- 끝나는 애니메이션만 `pumpAndSettle()`로 기다린다.
- 계속 흐르는 MQTT·BLE 스트림은 이벤트를 주입하고 `pump()`한다.
- 시간 기반 UI는 필요한 `Duration`만 흘린다.
- Fake 스트림과 타이머는 테스트 종료 시 반드시 정리한다.

`pumpAndSettle()`은 편리하지만 “화면이 안정될 때까지”라는 조건이 IoT 앱에서는 자주 성립하지 않는다. 연결 상태처럼 계속 변하는 화면일수록 기다리는 시간을 줄이는 것이 테스트 안정성을 높인다.
