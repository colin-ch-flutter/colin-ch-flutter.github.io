---
layout: post
title: "Flutter WidgetTester TextField 입력 테스트 - 스마트홈 폼 검증과 키보드 처리"
description: "Flutter WidgetTester에서 TextField 입력, Form validator, TextInputAction을 검증하고 pump 타이밍과 키보드 포커스 때문에 실패하는 스마트홈 제어 폼 테스트를 안정화하는 방법을 정리했다."
date: 2026-09-09
tags: [Flutter, Dart, 테스트, IoT, UX]
comments: true
share: true
---

![Flutter WidgetTester로 TextField와 스마트홈 폼 입력을 테스트하는 화면](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

이 그림에서 볼 부분은 화면 모양이 아니라, 사용자가 입력한 값이 validator를 통과하고 실제 명령 버튼까지 이어지는 흐름이다.

Flutter WidgetTester에서 TextField 테스트는 `enterText()` 한 줄이면 끝날 것 같지만, 실제로는 그렇지 않았다. 입력값이 바뀌었는데도 에러 문구가 그대로 남거나, 키보드가 열린 상태에서 버튼이 가려져 탭이 실패했다. 스마트홈 기기 이름과 목표 온도를 입력받는 폼은 입력·검증·포커스 해제를 각각 확인해야 재현성 있는 테스트가 된다.

## 입력 전과 입력 후를 분리한다

테스트 대상은 서버나 MQTT를 붙이지 않은 순수 폼으로 줄였다. `GlobalKey<FormState>`와 `TextEditingController`를 화면 내부에 두고, 유효한 입력일 때만 제출 콜백을 호출한다.

```dart
class DeviceForm extends StatefulWidget {
  const DeviceForm({super.key, required this.onSubmit});

  final ValueChanged<String> onSubmit;

  @override
  State<DeviceForm> createState() => _DeviceFormState();
}

class _DeviceFormState extends State<DeviceForm> {
  final _formKey = GlobalKey<FormState>();
  final _nameController = TextEditingController();

  @override
  void dispose() {
    _nameController.dispose();
    super.dispose();
  }

  void _submit() {
    FocusManager.instance.primaryFocus?.unfocus();
    if (_formKey.currentState!.validate()) {
      widget.onSubmit(_nameController.text.trim());
    }
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            key: const Key('device-name-field'),
            controller: _nameController,
            textInputAction: TextInputAction.done,
            validator: (value) => value == null || value.trim().length < 2
                ? '기기 이름을 2자 이상 입력하세요'
                : null,
            onFieldSubmitted: (_) => _submit(),
          ),
          ElevatedButton(
            key: const Key('device-submit-button'),
            onPressed: _submit,
            child: const Text('기기 저장'),
          ),
        ],
      ),
    );
  }
}
```

`enterText()` 뒤에 바로 `find.text()`를 호출하면 validator가 아직 실행되지 않은 상태를 검사하게 된다. 버튼을 탭하거나 `FormState.validate()`를 호출한 뒤 `pump()`를 한 번 진행해야 에러 문구가 위젯 트리에 반영된다.

```dart
testWidgets('빈 이름은 저장하지 않고 에러를 표시한다', (tester) async {
  String? submitted;
  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: DeviceForm(onSubmit: (value) => submitted = value),
      ),
    ),
  );

  await tester.tap(find.byKey(const Key('device-submit-button')));
  await tester.pump();

  expect(find.text('기기 이름을 2자 이상 입력하세요'), findsOneWidget);
  expect(submitted, isNull);

  await tester.enterText(
    find.byKey(const Key('device-name-field')),
    '거실 보일러',
  );
  await tester.testTextInput.receiveAction(TextInputAction.done);
  await tester.pump();

  expect(find.text('기기 이름을 2자 이상 입력하세요'), findsNothing);
  expect(submitted, '거실 보일러');
});
```

## 테스트가 흔들렸던 지점

| 상황 | 실패 원인 | 안정화 방법 |
| --- | --- | --- |
| 에러 문구가 안 보임 | validation 전 상태를 검사함 | 버튼 탭 뒤 `pump()` |
| 제출 콜백이 두 번 호출됨 | 버튼 탭과 `receiveAction`을 둘 다 제출 경로로 사용함 | 한 테스트에서는 경로 하나만 선택 |
| 키보드가 버튼을 가림 | 실제 포커스가 남아 있음 | 제출 시 `unfocus()` 또는 `viewInsets` 반영 |
| 입력값 비교가 실패함 | 앞뒤 공백을 그대로 전달함 | 저장 직전에 `trim()` |

처음엔 `pumpAndSettle()`을 습관처럼 넣었는데, 폼에 커서와 키보드가 남아 있으면 애니메이션이 끝나지 않아 테스트가 멈췄다. 이 화면처럼 검증 문구만 바뀌는 경우에는 `pump()`가 더 정확하다. 화면 전환이나 SnackBar까지 확인할 때만 필요한 만큼 `pump(const Duration(milliseconds: 200))`를 추가하는 편이 낫다.

짧게 정리하면, Flutter WidgetTester의 폼 테스트는 `enterText()` 자체보다 검증 시점과 제출 경로를 통제하는 일이 핵심이다. 빈 입력과 정상 입력을 한 테스트에서 구분하고, `pump()`와 포커스 해제를 명시하면 스마트홈 제어 화면의 입력 테스트가 키보드 상태에 덜 흔들린다.
