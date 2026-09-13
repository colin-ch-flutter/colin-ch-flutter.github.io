---
layout: post
title: "Flutter WidgetTester ExpansionTile 테스트 - 스마트홈 기기 상세 상태 검증"
description: "Flutter WidgetTester로 스마트홈 기기 카드의 ExpansionTile 펼침·접힘과 MQTT 상태 상세 정보를 검증하고, 애니메이션과 중복 텍스트 때문에 실패했던 테스트를 정리했다."
date: 2026-09-13
tags: [Flutter, Dart, 테스트, WidgetTester, MQTT, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 기기 ExpansionTile을 테스트하는 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter WidgetTester에서 `ExpansionTile`은 제목이 보이는지만 검사하면 부족하다. 스마트홈 기기 카드에서는 접힌 상태에서 핵심 상태를 보여주고, 펼쳤을 때 MQTT 수신 시각·신호 세기·마지막 명령 같은 상세 정보를 추가로 보여준다. 처음에는 `find.text('보일러')`만 확인했는데, 펼침 애니메이션이 끝나기 전에 검사를 실행하거나 같은 상태 문구가 두 곳에 있어 테스트가 쉽게 흔들렸다.

## 접힌 상태와 펼친 상태를 각각 정의한다

테스트 대상은 복잡할 필요가 없다. 중요한 점은 `ExpansionTile`에 테스트용 `Key`를 붙이고, 상세 영역에서만 나와야 하는 텍스트를 하나 정하는 것이다.

```dart
class DeviceTile extends StatelessWidget {
  const DeviceTile({super.key, required this.isOnline});

  final bool isOnline;

  @override
  Widget build(BuildContext context) {
    return ExpansionTile(
      key: const ValueKey('boiler-tile'),
      title: const Text('거실 보일러'),
      subtitle: Text(isOnline ? '온라인' : '오프라인'),
      children: [
        ListTile(
          key: const ValueKey('boiler-details'),
          title: const Text('MQTT 상세 상태'),
          subtitle: Text(isOnline ? '마지막 명령: 난방 시작' : '연결 대기 중'),
        ),
      ],
    );
  }
}
```

`boiler-details`는 접힌 순간에는 트리에 없고, 펼친 뒤에만 나타난다. 이 경계를 이용하면 단순히 글자가 화면에 존재하는지보다 실제 사용자 동작이 레이아웃에 반영됐는지를 확인할 수 있다.

## `tap` 뒤에는 필요한 만큼만 프레임을 진행한다

아래 테스트는 기기 타일의 초기 상태, 펼침, 다시 접힘을 한 흐름에서 검증한다. `pumpAndSettle()`은 ExpansionTile의 애니메이션이 끝날 때까지 기다리므로 이 경우에는 사용해도 괜찮지만, MQTT 스트림이나 반복 타이머가 함께 살아 있는 화면에서는 무한 대기가 될 수 있다.

```dart
testWidgets('기기 상세 상태를 펼치고 다시 접을 수 있다', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(
      home: Scaffold(
        body: DeviceTile(isOnline: true),
      ),
    ),
  );

  expect(find.byKey(const ValueKey('boiler-tile')), findsOneWidget);
  expect(find.byKey(const ValueKey('boiler-details')), findsNothing);
  expect(find.text('마지막 명령: 난방 시작'), findsNothing);

  await tester.tap(find.byKey(const ValueKey('boiler-tile')));
  await tester.pumpAndSettle();

  expect(find.byKey(const ValueKey('boiler-details')), findsOneWidget);
  expect(find.text('MQTT 상세 상태'), findsOneWidget);
  expect(find.text('마지막 명령: 난방 시작'), findsOneWidget);

  await tester.tap(find.byKey(const ValueKey('boiler-tile')));
  await tester.pumpAndSettle();

  expect(find.byKey(const ValueKey('boiler-details')), findsNothing);
});
```

여기서 `find.text('거실 보일러')`를 탭하지 않고 `Key`를 사용한 이유가 있다. 실제 카드에 상태 요약 위젯이나 접근성 라벨이 추가되면 같은 문자열이 두 번 매칭될 수 있다. `find.byKey`는 UI 문구가 바뀌어도 동작의 대상을 유지한다.

## MQTT 상태 갱신은 펼침 테스트와 분리한다

실제 앱에서는 타일을 펼친 뒤 MQTT 상태 스트림이 갱신된다. 처음에는 펼침 테스트 안에서 publish, ACK 수신, 화면 갱신까지 모두 확인했는데 실패 지점이 너무 많았다. 테스트 목적을 나누면 원인을 훨씬 빨리 찾을 수 있다.

| 검증 대상 | 테스트에서 확인할 것 | 실패하기 쉬운 지점 |
|---|---|---|
| 기본 카드 | 제목과 온라인 상태 | 중복 텍스트, 잘못된 `Finder` |
| 펼침 동작 | 상세 타일이 나타남 | 애니메이션 전 `expect` 실행 |
| MQTT 갱신 | 상세 문구가 새 값으로 변경됨 | Fake stream 미방출 |
| 접힘 동작 | 상세 타일이 사라짐 | 이전 프레임을 확인함 |

MQTT 갱신은 `FakeMqttService`가 상태를 발행한 뒤 `await tester.pump()`을 호출하는 별도 테스트로 두는 편이 낫다. `pumpAndSettle()`을 습관처럼 넣으면 연결 스트림이 계속 살아 있는 화면에서 테스트가 끝나지 않을 수 있다. 상태 한 번만 바뀌는 경우에는 `pump()` 한 번으로도 충분하다.

## 실제로 걸렸던 문제

내 테스트에서는 접힌 상태에서 `find.text('온라인')`이 통과해서 처음엔 문제가 없는 줄 알았다. 그런데 상세 영역에도 동일한 상태 배지를 넣자 `findsOneWidget`이 `findsNWidgets(2)`로 바뀌었다. 화면의 현재 문구를 검사할 때는 `findsOneWidget`을 무조건 고집하기보다, 어느 영역의 텍스트인지 `find.descendant`로 범위를 좁히는 편이 정확했다.

또 `ExpansionTile`의 `initiallyExpanded: true`를 테스트에 넣으면 펼침 동작 자체를 검증할 수 없다. 초기 상태를 명시적으로 접힌 상태로 두고, 사용자의 탭을 거친 뒤 상세 영역이 나타나는지 확인해야 한다.

솔직하게 정리하면, Flutter WidgetTester의 `ExpansionTile` 테스트 핵심은 애니메이션보다 상태 경계다. `Key`로 타일을 특정하고, 접힘·펼침·MQTT 갱신을 분리하고, 스트림이 있는 화면에서는 필요한 프레임만 진행하면 테스트가 안정된다.
