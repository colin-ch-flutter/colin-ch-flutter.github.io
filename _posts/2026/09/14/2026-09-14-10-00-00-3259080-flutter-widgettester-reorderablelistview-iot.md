---
layout: post
title: "Flutter WidgetTester ReorderableListView 테스트 - 스마트홈 자동화 순서 변경 검증"
description: "Flutter WidgetTester로 ReorderableListView의 드래그 순서 변경과 저장 콜백을 검증하고, Key 누락과 잘못된 드래그 좌표 때문에 실패한 테스트를 고치는 방법을 정리했다."
date: 2026-09-14
tags: [Flutter, Dart, 테스트, IoT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 자동화 순서를 검증하는 화면](https://images.unsplash.com/photo-1558002038-1055907df827?w=1200&q=80)

스마트홈 자동화 목록의 순서 변경은 화면에서 손으로 해보면 간단하지만, Flutter WidgetTester에서는 `Key`, 드래그 위치, 저장 콜백을 모두 맞춰야 제대로 검증된다. 이번에는 보일러 자동화 목록에서 항목을 끌어 순서를 바꾸고, 바뀐 목록이 저장 요청까지 전달되는지 테스트했다.

## 처음에는 텍스트 순서만 확인했다

처음 작성한 테스트는 `find.text('귀가 모드')`가 화면에 있는지만 확인했다. 이 검사는 목록이 그려졌다는 사실만 알려준다. 실제 버그는 드래그 직후 화면 순서는 바뀌었지만 Repository에 저장하는 콜백이 예전 순서를 보내는 경우였다.

스마트홈 자동화에서는 순서가 곧 실행 우선순위다. `난방 켜기`와 `환기 시작`의 위치가 뒤집히면 같은 화면이라도 기기 동작 결과가 달라진다. 그래서 이번 테스트의 기준을 아래처럼 잡았다.

| 검증 항목 | 확인 방법 |
| --- | --- |
| 목록 항목 식별 | 각 타일에 고유 `ValueKey` 부여 |
| 순서 변경 | `tester.drag` 후 `pumpAndSettle` |
| 화면 상태 | 첫 번째·마지막 항목의 텍스트 확인 |
| 저장 요청 | Fake Repository에 전달된 ID 순서 확인 |

## 항목마다 Key를 고정한다

드래그 대상이 재빌드된 뒤에도 같은 항목인지 알아야 하므로, 표시 순번이 아니라 자동화 ID를 `Key`로 사용했다. 순번을 키로 쓰면 항목을 옮긴 순간 키가 다른 데이터에 붙어 테스트와 상태가 꼬인다.

아래 위젯은 자동화 항목을 세로 목록으로 보여주고, 순서가 바뀌면 `onReordered`를 호출한다.

```dart
class AutomationList extends StatelessWidget {
  const AutomationList({
    required this.items,
    required this.onReordered,
    super.key,
  });

  final List<Automation> items;
  final ValueChanged<List<Automation>> onReordered;

  @override
  Widget build(BuildContext context) {
    return ReorderableListView.builder(
      itemCount: items.length,
      onReorder: (oldIndex, newIndex) {
        final updated = [...items];
        if (newIndex > oldIndex) newIndex -= 1;
        final item = updated.removeAt(oldIndex);
        updated.insert(newIndex, item);
        onReordered(updated);
      },
      itemBuilder: (context, index) {
        final item = items[index];
        return ListTile(
          key: ValueKey('automation-${item.id}'),
          title: Text(item.name),
          subtitle: Text(item.deviceName),
        );
      },
    );
  }
}
```

여기서 `newIndex > oldIndex`일 때 1을 빼는 처리가 빠지면 아래쪽으로 옮길 때 최종 위치가 한 칸 어긋난다. `ReorderableListView`가 새 위치를 계산하는 방식과 리스트에 직접 삽입하는 방식이 다르기 때문이다.

## WidgetTester로 드래그를 검증한다

테스트에서는 실제 네트워크나 MQTT를 연결하지 않고, 순서 저장만 기록하는 Fake Repository를 사용했다. 드래그 직후에는 애니메이션이 남을 수 있어 `pumpAndSettle`을 호출한다.

```dart
testWidgets('자동화 항목을 드래그하면 새 순서로 저장한다', (tester) async {
  final repository = FakeAutomationRepository([
    Automation(id: 'heat', name: '난방 켜기', deviceName: '거실 보일러'),
    Automation(id: 'air', name: '환기 시작', deviceName: '환풍기'),
    Automation(id: 'away', name: '외출 모드', deviceName: '전체 공간'),
  ]);

  await tester.pumpWidget(
    MaterialApp(
      home: AutomationPage(repository: repository),
    ),
  );
  await tester.pumpAndSettle();

  final source = find.byKey(const ValueKey('automation-away'));
  final target = find.byKey(const ValueKey('automation-heat'));

  await tester.drag(source, const Offset(0, -90));
  await tester.pumpAndSettle();

  expect(find.text('외출 모드'), findsOneWidget);
  expect(repository.lastSavedIds, ['away', 'heat', 'air']);
  expect(
    tester.getTopLeft(find.text('외출 모드')).dy,
    lessThan(tester.getTopLeft(target).dy),
  );
});
```

여기서 텍스트가 존재하는지만 보지 않고 Fake Repository의 `lastSavedIds`를 함께 검사한 이유는 화면과 데이터 레이어의 불일치를 잡기 위해서다. 실제로 첫 구현에서는 화면은 새 순서로 다시 그려졌지만, Controller가 원본 리스트를 Repository에 넘겨 저장하고 있었다. 눈으로는 통과처럼 보였고 앱을 다시 열었을 때만 순서가 되돌아갔다.

## 드래그 테스트가 자주 실패한 이유

드래그가 동작하지 않을 때 `pumpAndSettle`만 반복해서 추가하면 해결되지 않는다. 실패 원인은 대체로 세 가지였다.

| 증상 | 원인 | 수정 기준 |
| --- | --- | --- |
| 대상 항목을 못 찾음 | index 기반 Key 사용 | 기기·자동화 ID를 Key로 사용 |
| 순서가 한 칸 어긋남 | `newIndex` 보정 누락 | 아래 방향 이동 시 1 감소 |
| 드래그 후 저장 검증 실패 | UI 리스트만 갱신 | Fake Repository 저장값도 검사 |

특히 화면 높이가 작은 테스트 환경에서는 `Offset(0, -90)`이 충분하지 않을 수 있다. 고정 픽셀 하나에 의존하기보다 대상 항목의 위치를 읽어 이동 거리를 계산하는 방식이 더 안정적이다. 목록이 스크롤되는 구조라면 `dragUntilVisible` 또는 스크롤 컨테이너를 먼저 준비하는 흐름도 필요하다.

## 짧게 정리하면

Flutter WidgetTester에서 `ReorderableListView`를 테스트할 때는 드래그 성공 여부만 보지 말고, 고유 `Key`, `newIndex` 보정, 저장 레이어의 최종 순서를 함께 고정해야 한다. `find.text` 하나로 끝내면 화면만 바뀌고 실제 자동화 순서는 저장되지 않는 버그를 놓친다. IoT 앱의 순서 변경 UI는 작은 기능처럼 보여도 실행 결과에 직접 영향을 주므로, Fake Repository까지 연결한 Widget Test가 비용 대비 효과가 컸다.
