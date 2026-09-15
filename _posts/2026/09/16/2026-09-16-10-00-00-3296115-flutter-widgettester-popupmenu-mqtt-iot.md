---
layout: post
title: "Flutter WidgetTester PopupMenuButton 테스트 - 스마트홈 자동화 MQTT 메뉴 검증"
description: "Flutter WidgetTester로 PopupMenuButton 오버레이를 열고 스마트홈 자동화의 MQTT 실행·수정 메뉴를 검증하는 방법과 tap 대상이 여러 개일 때의 해결 기준을 정리했다."
date: 2026-09-16
tags: [Flutter, Dart, 테스트, WidgetTester, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester PopupMenuButton으로 스마트홈 자동화 메뉴를 테스트하는 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

PopupMenuButton은 화면에 버튼이 보인다고 테스트가 끝나지 않는다. 메뉴가 오버레이로 늦게 생기고, 같은 메뉴 문구가 여러 자동화 카드에 반복되기 때문이다. Flutter WidgetTester에서는 카드의 Key로 메뉴를 연 뒤, 실제 MQTT 명령을 선택했는지까지 확인해야 테스트가 덜 흔들린다.

## 처음 시도한 테스트가 실패한 이유

처음에는 `find.text('실행')`만 찾아 탭했다. 카드가 하나일 때는 통과했지만 자동화가 두 개가 되자 `Finder`가 2개를 반환했다. 더 답답했던 부분은 `PopupMenuButton`을 탭한 직후 메뉴 항목을 찾으려 한 것이다. 메뉴는 같은 위젯 트리에 자식으로 붙는 것이 아니라 Overlay에 추가되므로, 프레임을 한 번 진행해야 한다.

테스트 대상은 카드별 Key와 선택 값만 안정적으로 노출하도록 분리했다.

```dart
class AutomationCard extends StatelessWidget {
  const AutomationCard({required this.automation, required this.onAction, super.key});
  final Automation automation;
  final ValueChanged<String> onAction;

  @override
  Widget build(BuildContext context) {
    return Card(key: ValueKey('automation-${automation.id}'), child: ListTile(
      title: Text(automation.name),
      trailing: PopupMenuButton<String>(
        key: ValueKey('menu-${automation.id}'), onSelected: onAction,
        itemBuilder: (_) => const [
          PopupMenuItem(value: 'run', child: Text('실행')),
          PopupMenuItem(value: 'edit', child: Text('수정')),
        ],
      ),
    ));
  }
}
```

`value`를 문구와 분리하면 화면 문구를 바꿔도 Controller의 MQTT 명령은 유지된다.

## 오버레이를 열고 MQTT 명령을 검증한다

거실 자동화의 메뉴를 열고 `run` 값이 Fake Repository로 전달되는지 확인한다.

```dart
testWidgets('거실 자동화 메뉴에서 실행을 선택하면 MQTT 명령을 보낸다', (tester) async {
  final fakeRepository = FakeAutomationRepository();
  final controller = AutomationController(fakeRepository);

  await tester.pumpWidget(
    MaterialApp(
      home: AutomationPage(controller: controller),
    ),
  );

  await tester.tap(find.byKey(const ValueKey('menu-living-room')));
  await tester.pump();

  expect(find.byType(PopupMenuItem<String>), findsNWidgets(2));
  expect(find.text('실행'), findsOneWidget);

  await tester.tap(find.text('실행'));
  await tester.pump();

  expect(fakeRepository.commands, [
    const AutomationCommand(id: 'living-room', action: 'run'),
  ]);
});
```

첫 번째 `pump()`는 Overlay를 그릴 기회를 주고, 두 번째 호출은 선택 콜백 이후 프레임을 처리한다. MQTT 스트림처럼 계속 이벤트가 발생하면 `pumpAndSettle()`가 무한 대기할 수 있어 `pump()`를 명시했다.

## 메뉴가 닫히는 것도 상태다

명령을 보낸 뒤 메뉴가 닫혔는지까지 확인하면 같은 항목을 중복 탭하는 회귀를 잡을 수 있다.

```dart
expect(find.byType(PopupMenuItem<String>), findsNothing);
expect(find.text('실행 중'), findsOneWidget);
```

선택 직후 메뉴가 닫히는 것은 기본 동작이다. 실패하면 `pump()` 누락과 `onSelected` 예외를 먼저 확인한다. `showMenu`를 직접 썼다면 항목 개수보다 Key로 찾는 편이 안전하다.

| 검증 대상 | 권장 Finder | 이유 |
|---|---|---|
| 카드별 메뉴 버튼 | `byKey` | 자동화 이름 중복에 안전하다 |
| 오버레이 메뉴 표시 | `byType`·고유 문구 | 메뉴가 실제로 열렸는지 확인한다 |
| 선택 결과 | Fake Repository 기록 | MQTT 브로커 없이 명령을 검증한다 |
| 메뉴 닫힘 | `findsNothing` | 중복 실행 회귀를 잡는다 |

솔직하게 정리하면 `PopupMenuButton` 테스트의 핵심은 메뉴 문구를 찾는 기술이 아니라, Overlay가 생기는 프레임과 명령 전달 경계를 나누는 일이다. 카드마다 안정적인 Key를 두고, 선택 값은 UI 문구와 분리하고, MQTT는 Fake로 격리하면 자동화 카드가 2개에서 20개로 늘어나도 테스트 의도가 유지된다.
