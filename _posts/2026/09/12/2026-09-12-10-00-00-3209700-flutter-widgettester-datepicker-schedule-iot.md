---
layout: post
title: "Flutter WidgetTester showDatePicker 테스트 - 스마트홈 예약 날짜 검증"
description: "Flutter WidgetTester로 showDatePicker를 여는 예약 화면을 테스트하는 방법을 정리했다. 초기 날짜, 날짜 선택, 취소 후 상태 복원까지 검증한다."
date: 2026-09-12
tags: [Flutter, Dart, 테스트, IoT, 스마트홈]
comments: true
share: true
---

![Flutter 스마트홈 예약 날짜 선택 테스트 — 달력에서 고른 날짜가 화면에 반영되는 흐름](https://images.unsplash.com/photo-1506784983877-45594efa4cbe?w=1200&q=80)

Flutter WidgetTester에서 `showDatePicker`를 테스트할 때는 달력의 날짜를 찾는 것보다 **다이얼로그가 열린 뒤의 위젯 트리와 저장된 값의 변화를 나눠 검증하는 것**이 핵심이다. 스마트홈 예약 화면에서는 날짜를 골랐다는 사실만 확인하면 부족하다. 취소했을 때 기존 예약일이 유지되는지도 봐야 한다.

처음엔 `tester.tap(find.text('15'))`만 하면 끝날 줄 알았다. 해보니 현재 달과 이전·다음 달에 같은 숫자가 동시에 존재할 수 있어 테스트가 흔들렸다. 날짜를 고른 뒤 화면에 `2026. 9. 15.`가 표시되는지까지 확인하는 편이 안정적이다.

## 테스트 대상 화면을 작게 만든다

실제 앱에서는 Controller와 Repository가 붙어 있지만, 여기서는 날짜 선택 흐름만 격리한다. 버튼을 누르면 예약일을 선택하고, 취소하면 `selectedDate`를 바꾸지 않는다.

```dart
class ScheduleDateField extends StatefulWidget {
  const ScheduleDateField({super.key});

  @override
  State<ScheduleDateField> createState() => _ScheduleDateFieldState();
}

class _ScheduleDateFieldState extends State<ScheduleDateField> {
  DateTime selectedDate = DateTime(2026, 9, 12);

  Future<void> _pickDate() async {
    final picked = await showDatePicker(
      context: context,
      initialDate: selectedDate,
      firstDate: DateTime(2026, 1, 1),
      lastDate: DateTime(2026, 12, 31),
    );

    if (!mounted || picked == null) return;
    setState(() => selectedDate = picked);
  }

  @override
  Widget build(BuildContext context) {
    final label = '${selectedDate.year}. ${selectedDate.month}. '
        '${selectedDate.day}.';

    return Column(
      children: [
        Text(label, key: const Key('schedule-date')),
        ElevatedButton(
          onPressed: _pickDate,
          child: const Text('예약 날짜 선택'),
        ),
      ],
    );
  }
}
```

`showDatePicker`는 `Future<DateTime?>`를 반환한다. 그래서 버튼 탭 직후 바로 화면을 검사하면 안 된다. 다이얼로그가 표시될 때까지 한 번 펌프하고, 날짜를 누른 뒤 닫힐 때 다시 펌프해야 한다.

## 다이얼로그 열기와 날짜 선택

다음 테스트는 기본 예약일이 보이는지, 버튼 탭으로 날짜 선택기가 나타나는지, 15일을 고른 뒤 값이 바뀌는지를 확인한다.

```dart
testWidgets('예약 날짜를 선택하면 화면의 날짜가 바뀐다', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(home: Scaffold(body: ScheduleDateField())),
  );

  expect(find.byKey(const Key('schedule-date')), findsOneWidget);
  expect(find.text('2026. 9. 12.'), findsOneWidget);

  await tester.tap(find.text('예약 날짜 선택'));
  await tester.pumpAndSettle();

  expect(find.byType(DatePickerDialog), findsOneWidget);
  expect(find.text('2026'), findsOneWidget);

  await tester.tap(find.text('15', skipOffstage: false));
  await tester.pumpAndSettle();

  expect(find.byType(DatePickerDialog), findsNothing);
  expect(find.text('2026. 9. 15.'), findsOneWidget);
});
```

여기서 `skipOffstage: false`는 달력 내부의 날짜가 레이아웃 상태에 따라 offstage로 판단되는 경우를 대비한 옵션이다. 다만 숫자만 찾는 방식은 월 경계에서 여전히 모호할 수 있다. 앱의 최소 지원 Flutter 버전에서 접근성 라벨이 일정하다면 `find.bySemanticsLabel('15일')`처럼 의미 기반 Finder를 붙이는 편이 낫다.

## 취소하면 기존 날짜를 유지해야 한다

예약 화면에서 사용자가 달력을 열었다가 취소하는 일은 흔하다. 이때 임시 선택값을 곧바로 `selectedDate`에 넣으면 취소했는데도 날짜가 바뀌는 버그가 생긴다. 테스트에서는 `Cancel`을 누른 뒤 기존 날짜가 남아 있는지 확인한다.

```dart
testWidgets('날짜 선택을 취소하면 기존 예약일을 유지한다', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(home: Scaffold(body: ScheduleDateField())),
  );

  await tester.tap(find.text('예약 날짜 선택'));
  await tester.pumpAndSettle();

  await tester.tap(find.text('Cancel'));
  await tester.pumpAndSettle();

  expect(find.byType(DatePickerDialog), findsNothing);
  expect(find.text('2026. 9. 12.'), findsOneWidget);
});
```

한국어 로케일을 강제로 적용한 앱이라면 `Cancel`을 그대로 찾지 말아야 한다. `MaterialApp(localizationsDelegates: ..., supportedLocales: const [Locale('ko')])`를 사용하는 순간 버튼 텍스트가 달라질 수 있다. 이 경우에는 `find.byType(TextButton)`만으로 고르지 말고, 취소 버튼에 `Key('date-picker-cancel')`를 제공하거나 `find.bySemanticsLabel('취소')`를 사용한다.

## 테스트에서 실제로 걸러진 문제

| 검증 항목 | 놓치기 쉬운 실패 | 테스트 기준 |
| --- | --- | --- |
| 초기 날짜 | 오늘 날짜가 테스트 실행일에 따라 변함 | 고정된 `DateTime` 주입 |
| 날짜 선택 | 같은 숫자가 다른 달에도 존재함 | 월·연도와 결과 텍스트 함께 확인 |
| 취소 | 임시 선택값이 실제 상태에 반영됨 | 취소 뒤 기존 날짜 유지 |
| 로케일 | `Cancel`·날짜 포맷이 달라짐 | Key 또는 Semantics 사용 |

실제 IoT 앱에서 예약 명령은 날짜가 바뀐 뒤 MQTT payload를 만든다. 그래서 Widget Test에서는 publish 자체를 검증하기보다, 날짜 상태가 바뀐 시점에 화면 모델이 올바른 날짜를 갖는지 확인하고, payload 변환은 별도 단위 테스트로 나누는 게 깔끔하다.

솔직하게 정리하면, `showDatePicker` 테스트의 난점은 달력 UI가 아니라 날짜를 시스템 현재 시각에 묶어버리는 데 있다. 초기 날짜와 시간대를 고정하고, 선택 성공과 취소를 각각 검사하면 CI에서도 재현 가능한 예약 테스트가 된다.
