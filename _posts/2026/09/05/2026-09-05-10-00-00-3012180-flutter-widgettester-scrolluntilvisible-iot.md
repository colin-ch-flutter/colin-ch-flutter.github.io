---
layout: post
title: "Flutter WidgetTester 스크롤 테스트 - 긴 IoT 기기 목록에서 카드 찾기"
description: "Flutter WidgetTester와 dragUntilVisible로 긴 IoT 기기 목록을 스크롤하며 MQTT 상태 카드가 실제로 노출되는지 검증하고, 무한 스크롤 테스트가 불안정해지는 원인을 정리했다."
date: 2026-09-05
tags: [Flutter, Dart, 테스트, IoT, MQTT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester로 긴 IoT 기기 목록을 스크롤하는 테스트 화면](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

*이 그림에서 볼 것은 카드 디자인이 아니라, 화면 아래에 있는 기기도 사용자가 실제로 찾을 수 있는지 검증하는 흐름이다.*

Flutter WidgetTester에서 `find.text()`만 호출하면 처음 보이는 기기만 테스트하게 된다. 스마트홈 앱의 기기 목록이 20개를 넘으면 MQTT 상태 카드가 트리에 있어도 화면 밖에 있을 수 있다. 이번에는 `dragUntilVisible`로 목록을 움직인 뒤, 원하는 카드의 상태와 탭 동작을 검증했다.

## 화면에 있다고 테스트가 통과하는 것은 아니다

처음에는 `find.byKey(const ValueKey('boiler-18'))`가 찾아지면 성공이라고 생각했다. 그런데 Finder는 현재 viewport 밖에 있는 위젯도 찾는다. 화면에 렌더링됐다는 것과 사용자가 볼 수 있다는 것은 다른 조건이었다.

| 검사 방식 | 잡아내는 것 | 놓치는 것 |
|---|---|---|
| `find.byKey`만 사용 | 위젯 존재, 데이터 매핑 | 화면 밖에 있는 상태 |
| `tester.tap`만 사용 | 탭 콜백 연결 | 탭 대상이 viewport 밖인 경우 |
| `dragUntilVisible` 후 검사 | 실제 스크롤과 노출 | 서버가 계속 페이지를 추가하는 경우 |

고정된 테스트 데이터에서는 목록의 끝을 정하고, 그 안에서 스크롤 가능한지까지 확인하는 편이 안정적이다. 실제 MQTT 브로커를 붙이면 메시지 도착 타이밍이 섞이므로 화면 테스트에서는 `FakeDeviceRepository`를 사용했다.

## 기기 카드에 안정적인 Key를 부여한다

텍스트 대신 기기 ID를 Key로 사용해야 이름이나 다국어 문구가 바뀌어도 스크롤 테스트가 깨지지 않는다.

```dart
class DeviceList extends StatelessWidget {
  const DeviceList({required this.devices, super.key});

  final List<Device> devices;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      key: const ValueKey('device-list'),
      itemCount: devices.length,
      itemBuilder: (context, index) {
        final device = devices[index];
        return DeviceCard(
          key: ValueKey('device-${device.id}'),
          device: device,
        );
      },
    );
  }
}
```

카드 높이가 기기 상태에 따라 달라져도 Key가 있으면 목표 위젯을 다시 찾을 수 있다. 반대로 `find.text(device.name)`에 의존하면 같은 이름의 기기가 두 개일 때 어떤 카드를 찾았는지 알 수 없었다.

## dragUntilVisible로 실제 위치까지 이동한다

테스트에서는 30개의 가짜 기기를 만들고 18번째 보일러를 목록 아래에 배치했다. `dragUntilVisible`의 세 번째 인자는 한 번에 이동할 거리다. 너무 큰 값을 주면 overscroll 때문에 테스트가 플랫폼별로 흔들릴 수 있어 카드 높이보다 조금 큰 값으로 제한했다.

```dart
testWidgets('목록 아래의 보일러 카드를 스크롤해 MQTT 상태를 확인한다',
    (tester) async {
  final devices = List.generate(
    30,
    (index) => Device(
      id: '$index',
      name: index == 18 ? '안방 보일러' : '기기 $index',
      mqttState: index == 18 ? MqttState.online : MqttState.offline,
    ),
  );

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(body: DeviceList(devices: devices)),
    ),
  );

  final list = find.byKey(const ValueKey('device-list'));
  final target = find.byKey(const ValueKey('device-18'));

  expect(target, findsOneWidget);
  expect(tester.getRect(target).top, greaterThan(600));

  await tester.dragUntilVisible(target, list, 160);
  await tester.pumpAndSettle();

  expect(tester.getTopLeft(target).dy, greaterThanOrEqualTo(0));
  expect(tester.getBottomRight(target).dy, lessThanOrEqualTo(800));
  expect(find.text('온라인'), findsOneWidget);
});
```

`pumpAndSettle()`을 무조건 쓰면 반복 애니메이션 때문에 테스트가 끝나지 않을 수 있다. MQTT 연결 아이콘 테스트에서는 `await tester.pump(const Duration(milliseconds: 300));`처럼 필요한 시간만 진행했다.

## 무한 스크롤은 페이지 경계를 따로 검증한다

`NotificationListener<ScrollNotification>`에서 바닥에 닿으면 페이지를 요청하는 구조라면 `dragUntilVisible`만으로는 부족하다. Fake Repository가 두 번째 페이지를 반환하는지와 로딩 중 중복 요청을 막는지를 분리해서 검사해야 한다. 테스트 데이터가 부족한 상태에서 호출하면 목표를 영원히 찾는 것처럼 보이는 실패가 난다. `itemCount`와 Fake 데이터 개수를 맞추고, 페이지 로딩 테스트와 viewport 테스트를 나누니 디버깅이 쉬웠다.

## 짧게 정리하면

- Finder는 viewport 밖의 위젯도 찾으므로 화면 노출 검증과 분리한다.
- 기기 이름보다 고유 ID 기반 `ValueKey`를 사용한다.
- 고정 목록은 `dragUntilVisible`, 무한 목록은 페이지 로딩 테스트를 별도로 둔다.
- 반복 애니메이션이 있는 MQTT 카드는 무제한 `pumpAndSettle()`을 피한다.

긴 목록의 Widget Test는 스크롤 한 번을 흉내 내는 일이 아니다. 사용자가 목표 기기를 실제 화면에서 만날 수 있는지, 그 순간 최신 MQTT 상태가 보이는지를 함께 확인하는 테스트다.
