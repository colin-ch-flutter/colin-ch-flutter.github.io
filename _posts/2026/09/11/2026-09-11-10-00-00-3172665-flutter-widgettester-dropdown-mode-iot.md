---
layout: post
title: "Flutter WidgetTester DropdownButton 테스트 - 스마트홈 모드 선택과 오버레이 검증"
description: "Flutter WidgetTester로 스마트홈 DropdownButton의 메뉴 열기, 모드 선택, 화면 반영을 검증하고 오버레이 때문에 실패했던 테스트를 정리했다."
date: 2026-09-11
tags: [Flutter, Dart, 테스트, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 모드 선택 DropdownButton 테스트를 검증하는 화면](/assets/images/flutter-widgettester-dropdown-mode-iot.png)

이 그림에서 볼 부분은 화면 폭에 따라 달라지는 카드 배치와, 테스트가 실제 사용자처럼 모드 메뉴를 열고 선택하는 흐름이다.

Flutter WidgetTester에서 `DropdownButton`을 테스트할 때 현재 선택된 텍스트만 찾으면 충분하다고 생각하기 쉽다. 하지만 스마트홈 앱의 모드 선택은 메뉴를 열었는지, 올바른 항목을 골랐는지, 선택 뒤 카드 상태가 바뀌었는지까지 이어진다. 처음에는 `find.text('외출')`만 검사했는데, 메뉴 안의 항목과 화면에 표시된 현재 값이 동시에 찾아져서 테스트가 엉뚱한 위젯을 검증했다.

## 테스트 대상에 Key와 의미 있는 값을 둔다

MQTT 명령은 이 테스트에서 제외하고, 선택된 모드만 콜백으로 전달하는 작은 위젯을 만들었다. 오버레이 메뉴까지 포함한 화면 흐름을 검증하려면 테스트 전용 `Key`를 버튼에 고정하는 편이 디버깅하기 쉽다.

```dart
class ModeSelector extends StatelessWidget {
  const ModeSelector({super.key, required this.value, required this.onChanged});

  final String value;
  final ValueChanged<String> onChanged;

  @override
  Widget build(BuildContext context) {
    const modes = ['자동', '수동', '외출'];

    return DropdownButton<String>(
      key: const Key('mode-dropdown'),
      value: value,
      items: [
        for (final mode in modes)
          DropdownMenuItem(value: mode, child: Text(mode)),
      ],
      onChanged: (mode) {
        if (mode != null) onChanged(mode);
      },
    );
  }
}
```

여기서 `value`는 화면에 보이는 현재 모드이고, `DropdownMenuItem`의 `value`는 선택 이벤트로 전달되는 값이다. 둘을 같은 문자열로 맞추지 않으면 처음 렌더링부터 `There should be exactly one item` 예외가 나서 메뉴 테스트까지 도달하지 못한다.

## 메뉴를 연 뒤 항목을 범위 안에서 찾는다

아래 테스트는 버튼을 연 뒤 메뉴 항목을 선택하고, 콜백에 전달된 모드를 확인한다. 코드 바로 위에 `tap`과 `pump`를 나눈 이유는 메뉴가 오버레이에 올라오는 한 프레임을 명시적으로 진행하기 위해서다.

```dart
testWidgets('스마트홈 모드를 외출로 바꾼다', (tester) async {
  String? selectedMode = '자동';

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: ModeSelector(
          value: selectedMode!,
          onChanged: (mode) => selectedMode = mode,
        ),
      ),
    ),
  );

  expect(find.byKey(const Key('mode-dropdown')), findsOneWidget);
  expect(find.text('자동'), findsOneWidget);

  await tester.tap(find.byKey(const Key('mode-dropdown')));
  await tester.pumpAndSettle();

  expect(find.text('외출'), findsOneWidget);
  await tester.tap(find.text('외출').last);
  await tester.pump();

  expect(selectedMode, '외출');
});
```

실제 Controller가 상태를 보관한다면 콜백의 지역 변수 대신 Fake Controller를 주입하면 된다. 핵심은 메뉴를 열기 전의 `find.text`와 연 뒤의 `find.text`를 같은 의미로 취급하지 않는 것이다. 메뉴가 열린 상태에서는 현재 값과 오버레이의 항목이 함께 트리에 존재할 수 있다. 그래서 선택할 때는 `.last`를 쓰거나, 메뉴의 `ancestor` 범위를 좁혀야 한다.

| 확인 단계 | 테스트 기준 | 빠지기 쉬운 실패 |
| --- | --- | --- |
| 초기 화면 | `value`와 현재 라벨 | items의 value 불일치 |
| 메뉴 열기 | 오버레이 항목 존재 | `pump()` 누락 |
| 항목 선택 | 선택 콜백의 값 | 같은 텍스트를 잘못 탭함 |
| 화면 반영 | 새 모드의 카드 상태 | Controller 재빌드 누락 |

## `pumpAndSettle()`은 메뉴에서만 제한적으로 쓴다

처음에는 모든 단계에 `pumpAndSettle()`을 넣었다. 메뉴만 있는 샘플에서는 통과했지만, 실제 대시보드에는 MQTT 연결 상태를 갱신하는 Stream과 반복 Timer가 있었다. 그 화면에서 `pumpAndSettle()`은 끝나지 않거나, unrelated animation까지 기다리느라 테스트가 느려졌다.

메뉴가 나타나는 순간만 `pumpAndSettle()`로 처리하고, 선택 뒤 상태 갱신은 `pump()` 한 번으로 확인하는 방식이 더 안정적이었다. 애니메이션 자체를 검증하는 테스트가 아니라면 시간을 기다리는 것보다 상태가 바뀌는 경계를 직접 나누는 편이 낫다.

짧게 정리하면, Flutter WidgetTester의 `DropdownButton` 테스트는 현재 텍스트 검색만으로 끝내면 안 된다. `Key`로 버튼을 고정하고, 메뉴를 연 뒤 오버레이가 생기는 프레임을 진행하며, 중복 텍스트는 범위를 좁혀 선택해야 한다. MQTT나 BLE를 Fake로 격리하면 스마트홈 모드 선택 테스트도 네트워크 상태와 관계없이 빠르게 재현된다.
