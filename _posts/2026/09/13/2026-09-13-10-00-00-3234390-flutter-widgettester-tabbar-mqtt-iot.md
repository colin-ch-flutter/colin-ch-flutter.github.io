---
layout: post
title: "Flutter WidgetTester TabBar 테스트 - 스마트홈 MQTT 상태 탭 전환 검증"
description: "Flutter WidgetTester로 스마트홈 MQTT 대시보드의 TabBar를 테스트하고, 탭 인덱스·TabBarView·비동기 기기 상태 때문에 실패했던 검증 방법을 정리했다."
date: 2026-09-13
tags: [Flutter, Dart, 테스트, WidgetTester, MQTT, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 MQTT 대시보드의 탭 전환을 테스트하는 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter WidgetTester에서 `TabBar` 테스트는 탭 글자가 화면에 보이는지만 확인하면 부족하다. 스마트홈 앱에서는 ‘기기 상태’와 ‘예약’ 탭이 서로 다른 `TabBarView`를 보여주고, MQTT 스트림도 선택된 화면에 맞춰 표시한다. 처음에는 `find.text('예약')`를 찾은 뒤 바로 상태를 검사했는데, 실제로는 탭 인덱스만 바뀌고 애니메이션 프레임이 끝나지 않아 테스트가 간헐적으로 실패했다.

## 탭 자체와 화면 내용을 분리해 확인한다

테스트 대상은 실제 MQTT 브로커가 아니라 탭 전환 계약이다. 브로커 연결을 함께 넣으면 네트워크 지연과 탭 애니메이션 중 어느 쪽이 실패했는지 알기 어렵다. 화면은 선택된 탭에 따라 `Key`가 있는 본문을 렌더링하도록 단순화했다.

```dart
class SmartHomeTabs extends StatelessWidget {
  const SmartHomeTabs({super.key, required this.onTabChanged});

  final ValueChanged<int> onTabChanged;

  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 2,
      child: Builder(
        builder: (context) => Column(
          children: [
            TabBar(
              onTap: onTabChanged,
              tabs: const [
                Tab(key: Key('devices-tab'), text: '기기 상태'),
                Tab(key: Key('schedule-tab'), text: '예약'),
              ],
            ),
            const Expanded(
              child: TabBarView(
                children: [
                  Center(child: Text('MQTT 연결 기기', key: Key('device-view'))),
                  Center(child: Text('난방 예약', key: Key('schedule-view'))),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

여기서 `onTap`은 탭을 눌렀다는 이벤트만 알려준다. 실제 본문은 `TabBarView` 애니메이션을 거쳐 바뀌므로, 콜백이 호출됐다는 사실과 새 화면이 나타났다는 사실을 같은 expect로 처리하지 않는 편이 안전하다.

## `tap → pump → pumpAndSettle`을 구분한다

아래 테스트는 MQTT 서비스 대신 선택된 탭 인덱스만 기록한다. `pump()`는 탭 입력을 반영하고, `pumpAndSettle()`은 기본 탭 전환 애니메이션이 끝날 때까지 프레임을 진행한다.

```dart
testWidgets('스마트홈 대시보드에서 예약 탭으로 전환한다', (tester) async {
  final selectedIndexes = <int>[];

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: SmartHomeTabs(
          onTabChanged: selectedIndexes.add,
        ),
      ),
    ),
  );

  expect(find.byKey(const Key('device-view')), findsOneWidget);
  expect(find.byKey(const Key('schedule-view')), findsNothing);

  await tester.tap(find.byKey(const Key('schedule-tab')));
  await tester.pump();
  await tester.pumpAndSettle();

  expect(selectedIndexes, [1]);
  expect(find.byKey(const Key('schedule-view')), findsOneWidget);
  expect(find.byKey(const Key('device-view')), findsNothing);
});
```

`pumpAndSettle()`을 무조건 쓰는 것도 답은 아니다. 탭 화면 안에서 MQTT 구독처럼 계속 살아 있는 애니메이션이나 주기적 타이머가 실행되면 settle되지 않아 테스트가 멈출 수 있다. 그런 화면은 다음처럼 고정 시간만 진행하고, 본문이 바뀌었는지 검사한다.

```dart
await tester.tap(find.byKey(const Key('schedule-tab')));
await tester.pump(const Duration(milliseconds: 350));

expect(find.byKey(const Key('schedule-view')), findsOneWidget);
```

## 실제로 헷갈렸던 실패 지점

| 증상 | 원인 | 수정 기준 |
| --- | --- | --- |
| 탭 콜백은 호출됐는데 본문이 이전 화면이다 | 전환 애니메이션 프레임 미진행 | `pump()` 후 `pumpAndSettle()` 또는 고정 시간 `pump()` |
| `pumpAndSettle()`이 끝나지 않는다 | 반복 타이머·MQTT 스트림이 계속 동작 | Fake 시간 또는 제한된 `pump(Duration)` 사용 |
| 같은 텍스트가 두 개 발견된다 | 탭 라벨과 본문 문구가 중복됨 | `Key` 또는 `find.byType(Tab)`로 범위 축소 |
| 첫 탭에서 테스트가 불안정하다 | `DefaultTabController`를 만든 직후 상태 검사 | 초기 `pumpWidget` 뒤 현재 탭을 명시적으로 확인 |

탭 테스트에서 가장 큰 실수는 화면 전환을 네트워크 테스트처럼 다루는 것이다. MQTT 메시지가 도착했을 때 선택된 탭에 상태가 표시되는지까지 검증하려면, Fake 스트림이 이벤트를 한 번만 내보내도록 만들고 탭 전환 테스트와 상태 반영 테스트를 분리하는 편이 결과를 읽기 쉽다.

짧게 정리하면, `TabBar`는 라벨이 아니라 탭 인덱스와 `TabBarView`의 결과를 함께 확인해야 한다. 애니메이션이 있는 기본 화면은 `pumpAndSettle()`, 계속 움직이는 MQTT 화면은 제한된 `pump(Duration)`을 선택하면 실패 원인이 훨씬 선명해진다.
