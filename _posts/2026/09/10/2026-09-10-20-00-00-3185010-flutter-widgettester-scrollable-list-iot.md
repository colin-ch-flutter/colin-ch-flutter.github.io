---
layout: post
title: "Flutter WidgetTester 스크롤 테스트 - 스마트홈 기기 목록 dragUntilVisible 검증"
description: "Flutter WidgetTester로 스마트홈 기기 목록을 스크롤하면서 보이지 않는 기기를 찾는 방법과 dragUntilVisible, ensureVisible, pump 타이밍 때문에 실패했던 테스트를 정리했다."
date: 2026-09-10
tags: [Flutter, Dart, 테스트, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 기기 목록 스크롤 테스트를 검증하는 화면](/assets/images/flutter-widgettester-scrollable-list-iot.png)

이 그림에서 볼 부분은 카드 디자인이 아니라, 화면 아래에 있는 기기까지 테스트가 실제 스크롤로 찾아가는 흐름이다.

Flutter WidgetTester에서 긴 스마트홈 기기 목록을 테스트할 때 `find.text('안방 보일러')`가 실패하면 기기가 없는 줄 알기 쉽다. 실제로는 ListView가 아직 해당 아이템을 빌드하지 않았을 뿐이었다. 처음엔 목록 데이터를 3개로 줄여 테스트했는데, 운영 화면처럼 30개를 넣자 화면에 보이지 않는 카드 검증이 전부 흔들렸다.

## 보이지 않는 위젯은 먼저 화면으로 가져온다

테스트 대상은 MQTT를 붙이지 않은 기기 목록으로 줄였다. 중요한 건 카드마다 안정적인 Key를 주고, 스크롤 가능한 영역을 `ListView` 하나로 명확하게 만드는 것이다.

```dart
class DeviceList extends StatelessWidget {
  const DeviceList({super.key, required this.devices});

  final List<String> devices;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      key: const Key('device-list'),
      itemCount: devices.length,
      itemBuilder: (context, index) {
        return ListTile(
          key: Key('device-${index + 1}'),
          title: Text(devices[index]),
          subtitle: const Text('연결됨'),
        );
      },
    );
  }
}
```

코드 바로 아래 테스트에서 목록의 마지막 항목을 찾기 전에 스크롤 위치를 직접 바꾼다. `dragUntilVisible`은 지정한 Finder가 보일 때까지 같은 방향으로 드래그하므로 긴 목록에 적합하다.

```dart
testWidgets('스크롤하면 마지막 스마트홈 기기를 검증한다', (tester) async {
  final devices = List.generate(30, (index) => '기기 ${index + 1}');

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(body: DeviceList(devices: devices)),
    ),
  );

  final lastDevice = find.byKey(const Key('device-30'));
  expect(lastDevice, findsNothing);

  await tester.dragUntilVisible(
    lastDevice,
    find.byKey(const Key('device-list')),
    const Offset(0, -300),
  );
  await tester.pump();

  expect(lastDevice, findsOneWidget);
  expect(find.text('기기 30'), findsOneWidget);
});
```

여기서 `findsNothing`은 데이터가 없다는 뜻이 아니다. 현재 viewport 밖이라 lazy build 대상이 아니라는 뜻에 가깝다. 이 차이를 몰라서 처음에는 `itemCount`와 fixture를 의심했다. `dragUntilVisible` 뒤에는 별도 `pumpAndSettle()`보다 `pump()`가 충분했다. 스크롤 애니메이션을 기다리는 테스트가 아니기 때문이다.

## dragUntilVisible과 ensureVisible은 쓰임이 다르다

이미 위젯이 트리에 만들어져 있지만 화면 밖에 있는 경우에는 `ensureVisible`이 더 직접적이다. 반대로 ListView.builder가 아직 해당 항목을 빌드하지 않았다면 먼저 `dragUntilVisible`로 찾아야 한다.

| 상황 | 선택할 API | 이유 |
| --- | --- | --- |
| lazy list의 먼 항목 | `dragUntilVisible` | 스크롤하며 위젯을 빌드함 |
| 이미 존재하는 위젯 | `ensureVisible` | 대상 위치로 바로 이동 |
| 단순 초기 렌더링 확인 | `findsOneWidget` | 스크롤 동작을 검증하지 않음 |
| 스크롤 뒤 상태 반영 | `pump()` | 한 프레임만 진행해 타이밍을 통제함 |

실제 화면에서 자주 쓰는 `ScrollController`를 테스트에 직접 주입하는 방법도 있다. 다만 컨트롤러의 `jumpTo()`는 사용자의 드래그 동작을 검증하지 않는다. 목록이 스크롤되는지 확인하는 테스트라면 `dragUntilVisible`이 더 실제 사용에 가깝다.

## 테스트가 실패했던 지점

첫 번째 실패는 `dragUntilVisible`의 두 번째 인자로 `find.byType(ListView)`를 넘겼을 때였다. 화면에 ListView가 두 개 들어간 레이아웃에서는 어느 스크롤 영역을 움직일지 모호해졌다. `Key('device-list')`를 붙이니 대상이 고정됐다.

두 번째는 드래그 거리를 너무 크게 잡은 경우다. `Offset(0, -1000)`은 빠르지만 중간 아이템의 노출 상태를 확인할 수 없고, 작은 테스트 화면에서는 경계값에 따라 마지막 항목이 아직 완전히 보이지 않을 수 있다. 300~400 정도의 일정한 이동량이 디버깅하기 편했다.

짧게 정리하면, Flutter WidgetTester의 스크롤 테스트는 데이터 존재와 화면 노출을 분리해서 검증해야 한다. lazy list의 먼 항목은 `dragUntilVisible`, 이미 빌드된 항목은 `ensureVisible`을 쓰고, 스크롤 영역에는 명시적인 Key를 준다. 이 세 가지를 지키면 스마트홈 기기 수가 늘어나도 목록 테스트가 `findsNothing` 오판으로 무너지지 않는다.
