---
layout: post
title: "Flutter WidgetTester Dismissible 테스트 - 스마트홈 자동화 삭제와 실행 취소 검증"
description: "Flutter WidgetTester로 Dismissible 기반 스마트홈 자동화 삭제를 검증하는 방법을 정리했다. Key 고정, 스와이프 임계값, MQTT 명령 미발행과 실행 취소까지 실제 실패 사례로 설명한다."
date: 2026-09-15
tags: [Flutter, Dart, 테스트, WidgetTester, MQTT, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 자동화 삭제를 테스트하는 화면](https://images.unsplash.com/photo-1558002038-1055907df827?w=1200&q=80)

Flutter WidgetTester에서 `Dismissible`을 테스트할 때는 항목이 사라졌다는 사실만 보면 부족하다. 스마트홈 자동화 목록에서는 삭제가 저장 요청으로 이어지는지, SnackBar 실행 취소로 원래 순서가 복구되는지 확인해야 한다. Fake Repository 호출과 MQTT 발행도 분리했다.

## 화면에서 삭제와 실행을 분리한다

처음에는 자동화 카드 전체를 `Dismissible`로 감쌌다. 카드 안의 “지금 실행” 버튼에서도 드래그가 시작됐고, 삭제 콜백에서 MQTT 명령을 발행하는 실수도 생겼다. 삭제는 저장소 작업이고, 자동화 실행은 별도의 사용자 행동이다.

코드에서는 자동화 ID를 `Dismissible`의 Key로 고정하고, 삭제 콜백은 저장소에만 연결했다.

```dart
Dismissible(
  key: ValueKey('automation-${item.id}'),
  direction: DismissDirection.endToStart,
  confirmDismiss: (_) => _confirm(context, item),
  onDismissed: (_) => onDelete(item),
  child: AutomationTile(item: item),
)
```

항목 위치가 바뀌어도 같은 대상을 찾으려면 index 기반 Key를 쓰면 안 된다. `automation-${item.id}`처럼 DB에서 유지되는 ID를 넣었다. `confirmDismiss`는 삭제 확인 다이얼로그와도 분리된다.

## WidgetTester로 스와이프를 재현한다

보일러 자동화를 지운 뒤 Repository에는 삭제가 한 번 기록되고 MQTT 명령은 발행되지 않는지 검사한다. 충분한 거리를 이동해야 `onDismissed`가 호출된다.

```dart
testWidgets('자동화 삭제는 MQTT를 발행하지 않는다', (tester) async {
  final repository = FakeAutomationRepository(
    initial: [Automation(id: 'boiler-1', name: '퇴근 후 난방')],
  );
  final mqtt = FakeMqttPublisher();
  await tester.pumpWidget(
    MaterialApp(home: AutomationPage(repository: repository, mqtt: mqtt)),
  );

  final tile = find.byKey(const ValueKey('automation-boiler-1'));
  await tester.drag(tile, const Offset(-420, 0));
  await tester.pumpAndSettle();

  expect(find.text('퇴근 후 난방'), findsNothing);
  expect(repository.deletedIds, ['boiler-1']);
  expect(mqtt.publishedTopics, isEmpty);
});
```

실패했던 지점은 제목 텍스트만 드래그한 부분이다. 텍스트는 타일의 일부라 대상이 작고, 애니메이션 전에 `deletedIds`를 검사하면 콜백이 아직 실행되지 않는다. `Key`로 잡고 `pumpAndSettle()` 뒤에 검사했다.

## 실행 취소는 삭제 결과와 분리해 검사한다

삭제 직후 SnackBar 실행 취소는 별도 시나리오로 둔다. 서버 반영 뒤 복구 API를 호출한다면 `deletedIds`와 복구 ID를 함께 확인한다.

| 검증 대상 | 테스트 기준 | 놓치기 쉬운 오류 |
| --- | --- | --- |
| 올바른 카드 | 안정적인 `ValueKey` | index Key 재사용 |
| 삭제 확정 | `drag` 후 `onDismissed` | 이동 거리 부족 |
| MQTT 안전성 | 발행 토픽이 비어 있음 | 삭제를 기기 명령으로 처리 |
| 실행 취소 | 복구 ID와 목록 복원 | SnackBar 전 프레임 검사 |

솔직하게 정리하면, `Dismissible` 테스트의 핵심은 스와이프 자체가 아니다. 삭제 ID, MQTT 미발행, 애니메이션 뒤 저장 상태를 함께 고정해야 한다.
