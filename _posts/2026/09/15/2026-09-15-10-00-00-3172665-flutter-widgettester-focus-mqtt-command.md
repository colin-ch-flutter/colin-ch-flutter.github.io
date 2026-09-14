---
layout: post
title: "Flutter WidgetTester 포커스 테스트 - 스마트홈 MQTT 명령 입력 검증"
description: "Flutter WidgetTester로 스마트홈 MQTT 명령 입력창의 포커스 이동과 키보드 제출을 검증하고, 화면은 정상인데 명령이 두 번 전송되던 테스트 실패 원인을 정리했다."
date: 2026-09-15
tags: [Flutter, Dart, 테스트, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 MQTT 명령 입력과 포커스 흐름 검증](/images/2026-09-15-flutter-widgettester-focus-mqtt.png)

Flutter WidgetTester로 스마트홈 MQTT 명령 입력창을 테스트할 때는 텍스트가 보이는지만 확인하면 부족하다. `TextField`에 포커스가 들어오는지, 키보드의 완료 액션이 한 번만 publish를 호출하는지까지 검증해야 실제 제어 흐름이 안전해진다.

그림에서 봐야 할 부분은 입력창의 포커스 테두리와 전송 버튼이다. 사용자는 둘 중 하나로 명령을 보낼 수 있지만, 테스트에서는 두 경로가 같은 콜백을 한 번만 실행하는지 확인해야 한다.

## 화면은 통과했는데 MQTT 명령이 두 번 나갔다

처음에는 다음 정도로 끝냈다.

```dart
expect(find.byKey(const Key('mqtt-command-input')), findsOneWidget);
expect(find.text('보일러 켜기'), findsOneWidget);
```

화면 회귀 테스트로는 맞다. 그런데 실제 기기에서 키보드 완료를 누르면 publish가 두 번 호출됐다. `onSubmitted`에서 전송하고, 포커스가 해제될 때 실행되는 공통 저장 로직에서도 같은 명령을 전송하고 있었기 때문이다. 화면에는 아무 이상이 없어서 WidgetTester를 보강하기 전까지 놓쳤다.

입력창과 버튼은 같은 `submitCommand`를 호출하되, 빈 문자열과 중복 제출을 한 곳에서 막도록 구조를 줄였다.

```dart
class MqttCommandPanel extends StatefulWidget {
  const MqttCommandPanel({required this.onPublish, super.key});

  final Future<void> Function(String command) onPublish;

  @override
  State<MqttCommandPanel> createState() => _MqttCommandPanelState();
}

class _MqttCommandPanelState extends State<MqttCommandPanel> {
  final _controller = TextEditingController();
  final _focusNode = FocusNode();
  bool _sending = false;

  Future<void> _submitCommand([String? value]) async {
    if (_sending) return;
    final command = (value ?? _controller.text).trim();
    if (command.isEmpty) return;

    setState(() => _sending = true);
    try {
      await widget.onPublish(command);
      _controller.clear();
      _focusNode.unfocus();
    } finally {
      if (mounted) setState(() => _sending = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        Expanded(
          child: TextField(
            key: const Key('mqtt-command-input'),
            controller: _controller,
            focusNode: _focusNode,
            textInputAction: TextInputAction.done,
            onSubmitted: _submitCommand,
          ),
        ),
        IconButton(
          key: const Key('mqtt-command-submit'),
          onPressed: _sending ? null : _submitCommand,
          icon: const Icon(Icons.send),
        ),
      ],
    );
  }
}
```

## 포커스와 키보드 제출을 따로 검증한다

이제 Fake publish 함수를 두고 입력창 포커스, 입력값, 키보드 제출 순서를 확인한다. `enterText`만 호출하면 포커스 동작을 놓치므로 `tap`을 먼저 넣었다.

```dart
testWidgets('MQTT 명령 입력창은 포커스를 받고 완료 시 한 번 publish한다',
    (tester) async {
  final published = <String>[];

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: MqttCommandPanel(
          onPublish: (command) async => published.add(command),
        ),
      ),
    ),
  );

  final input = find.byKey(const Key('mqtt-command-input'));
  await tester.tap(input);
  await tester.pump();
  expect(tester.binding.focusManager.primaryFocus, isNotNull);

  await tester.enterText(input, '  boiler/on  ');
  await tester.testTextInput.receiveAction(TextInputAction.done);
  await tester.pump();

  expect(published, ['boiler/on']);
  expect(find.text('boiler/on'), findsNothing);
});
```

여기서 `receiveAction`을 빼고 버튼만 누르면 키보드 경로가 테스트되지 않는다. 반대로 버튼 경로도 별도 테스트해야 한다. 두 테스트가 모두 같은 Fake에 한 번만 기록되는지 보면 중복 publish 회귀를 쉽게 잡을 수 있다.

| 검증 대상 | 놓치기 쉬운 실패 | 테스트 기준 |
| --- | --- | --- |
| 입력창 포커스 | 첫 입력이 키보드로 전달되지 않음 | `tap` 후 `primaryFocus` 확인 |
| 공백 처리 | MQTT 토픽 앞뒤 공백 전송 | publish 인자에 trim 결과 사용 |
| 키보드 완료 | `onSubmitted` 미연결 | `receiveAction(TextInputAction.done)` |
| 빠른 연속 제출 | 같은 명령이 중복 publish | `_sending` 동안 콜백 차단 |
| 버튼 제출 | 키보드만 고치고 버튼은 누락 | 버튼 Finder로 별도 tap |

## `pumpAndSettle`을 무조건 쓰지 않은 이유

MQTT 연결 상태를 실제 Stream으로 붙인 테스트에서 `pumpAndSettle`을 사용했더니 테스트가 끝나지 않았다. 연결 감시 Stream이 계속 살아 있기 때문이다. 이 위젯은 제출 콜백이 Fake에서 즉시 끝나므로 `pump()` 한 번이면 충분했다.

실제 네트워크 연결은 이 테스트 범위에서 뺐다. WidgetTester는 입력 이벤트와 화면 상태를 빠르게 확인하고, publish 성공·재연결·오프라인 큐는 Repository 또는 integration_test에서 따로 검증하는 편이 실패 원인을 좁히기 쉽다. 테스트가 느려진다고 네트워크 대기를 넣으면 포커스 버그를 찾는 테스트가 MQTT 상태에 끌려간다.

솔직하게 정리하면:

- `find.text`는 입력창이 있는지만 보장하고 포커스는 보장하지 않는다.
- 키보드 제출과 버튼 제출은 같은 함수로 모으되, 각각의 경로를 테스트한다.
- `trim`, 빈 값, 연속 제출 차단은 UI가 아니라 명령 전송 경계에서 고정한다.
- WidgetTester에는 Fake publish를 넣고 MQTT 브로커 연결은 테스트 밖으로 분리한다.

해보니 포커스 테스트는 접근성만을 위한 검사가 아니었다. 스마트홈처럼 한 번의 publish가 실제 기기 상태를 바꾸는 화면에서는 입력 이벤트의 순서와 호출 횟수 자체가 기능 계약이다.
