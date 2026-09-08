---
layout: post
title: "Flutter WidgetTester ScaffoldMessenger 테스트 - MQTT 명령 SnackBar 중복 방지"
description: "Flutter WidgetTester에서 ScaffoldMessenger로 표시하는 MQTT 명령 성공·실패 SnackBar를 검증하고, pump 타이밍과 중복 알림 문제를 해결하는 방법을 정리했다."
date: 2026-09-09
tags: [Flutter, Dart, MQTT, IoT, 테스트]
comments: true
share: true
---

![Flutter WidgetTester ScaffoldMessenger로 MQTT 명령 결과를 테스트하는 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

이 그림에서 볼 부분은 기기 명령 자체가 아니라, 명령 결과를 사용자에게 한 번만 알리는 피드백 영역이다.

Flutter IoT 화면에서 `SnackBar`를 확인할 때 `find.text()`만 호출하면 충분하다고 생각하기 쉽다. 실제로는 `ScaffoldMessenger`가 올라오기 전이거나, `pump()`를 호출하지 않아 테스트 트리에 메시지가 아직 없거나, 같은 MQTT 이벤트가 두 번 들어와도 테스트가 통과하는 문제가 생긴다. `ScaffoldMessenger`를 테스트 화면에 명시하고, 이벤트와 프레임 진행을 나눠야 성공·실패 피드백을 안정적으로 검증할 수 있다.

## 화면 안에 ScaffoldMessenger를 직접 둔다

아래 위젯은 MQTT 서비스의 결과를 받은 뒤 한 번만 SnackBar를 띄우는 최소 예다. 실제 프로젝트에서는 Controller가 `CommandResult`를 발행하고 화면이 그 결과를 구독하는 구조를 사용했다.

```dart
class CommandResultView extends StatelessWidget {
  const CommandResultView({super.key, required this.result});

  final CommandResult? result;

  void _showResult(BuildContext context) {
    final current = result;
    if (current == null) return;

    final messenger = ScaffoldMessenger.of(context);
    messenger
      ..hideCurrentSnackBar()
      ..showSnackBar(
        SnackBar(
          content: Text(
            current.isSuccess ? '보일러 명령이 적용됐습니다' : '명령 적용에 실패했습니다',
          ),
        ),
      );
  }

  @override
  Widget build(BuildContext context) {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _showResult(context);
    });

    return const Text('보일러 제어');
  }
}
```

`build()`에서 부수효과를 직접 실행하면 rebuild마다 SnackBar가 다시 생길 수 있다. 위 코드는 설명을 위한 축약 예시고, 실제 구현에서는 Controller의 결과를 `StatefulWidget`의 `didUpdateWidget`이나 Riverpod `ref.listen`처럼 한 번만 실행되는 경계에서 처리해야 한다. 핵심은 `hideCurrentSnackBar()`로 이전 메시지를 정리하고, 화면에는 현재 결과만 남기는 것이다.

## WidgetTester에서는 pump 순서를 분리한다

테스트에서 필요한 건 “버튼을 눌렀다”와 “MQTT ACK를 받았다” 사이의 시간 제어다. Fake 서비스가 결과를 반환한 뒤 한 프레임을 진행하고 SnackBar를 찾는다.

```dart
testWidgets('MQTT 명령 성공 결과를 SnackBar로 한 번 알린다', (tester) async {
  final fakeService = FakeMqttService(
    result: const CommandResult.success(),
  );

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: CommandPage(service: fakeService),
      ),
    ),
  );

  await tester.tap(find.text('보일러 켜기'));
  await tester.pump(); // 명령 발행과 ACK 처리

  expect(find.text('보일러 명령이 적용됐습니다'), findsOneWidget);
  expect(find.byType(SnackBar), findsOneWidget);

  await tester.pump(const Duration(seconds: 4));
  expect(find.byType(SnackBar), findsNothing);
});
```

`tap()` 뒤에 바로 `expect()`를 쓰면 안 된다. 탭 콜백에서 `await service.publish()`가 실행되고, 완료 후 상태가 바뀌기 때문이다. 반대로 `pumpAndSettle()`만 쓰면 SnackBar의 표시 시간까지 모두 기다리므로 테스트가 느려지고, MQTT 상태 스트림이 계속 살아 있을 때는 끝나지 않는다.

| 상황 | 테스트 호출 | 확인할 것 |
| --- | --- | --- |
| 명령 버튼 탭 | `tap()` 후 `pump()` | ACK 전송 로직 실행 |
| 성공·실패 메시지 표시 | `find.text()` | 올바른 문구 1개 |
| 연속 이벤트 | `pump()` 후 개수 확인 | SnackBar 중복 여부 |
| 표시 시간 종료 | `pump(Duration)` | 화면에서 제거되는지 |

## 중복 이벤트 테스트를 따로 둔다

스마트홈에서는 같은 ACK가 재전송될 수 있다. 처음에는 성공 메시지가 보이는지만 검사했는데, MQTT reconnect 뒤 같은 `commandId`의 ACK가 두 번 들어오자 SnackBar도 두 개가 쌓였다. 테스트에 중복 이벤트를 넣고 기존 메시지를 교체하는지 확인해야 한다.

```dart
testWidgets('같은 MQTT commandId의 ACK는 SnackBar를 중복 표시하지 않는다',
    (tester) async {
  final controller = FakeCommandController();

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: CommandPage(controller: controller),
      ),
    ),
  );

  controller.emit(const CommandResult.success(commandId: 'cmd-7'));
  await tester.pump();
  controller.emit(const CommandResult.success(commandId: 'cmd-7'));
  await tester.pump();

  expect(find.byType(SnackBar), findsOneWidget);
});
```

실제 Controller에는 마지막으로 처리한 `commandId`를 저장하고 같은 값이면 UI 이벤트를 건너뛰는 조건을 넣었다. 모든 성공 결과를 막으면 다른 명령의 피드백까지 사라지므로, 단순히 “최근 결과가 성공이면 무시”하는 식으로 구현하면 안 된다.

솔직하게 정리하면, Flutter WidgetTester에서 SnackBar 테스트의 핵심은 메시지 문자열보다 이벤트 수명이다. `ScaffoldMessenger`가 있는 테스트 환경을 만들고, `tap → pump → 결과 확인` 순서를 고정하고, MQTT ACK의 `commandId` 중복까지 검증해야 실제 앱의 알림 중복을 잡을 수 있다.
