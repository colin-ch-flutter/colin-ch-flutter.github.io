---
layout: post
title: "Flutter WidgetTester ModalBottomSheet 테스트 - 스마트홈 MQTT 명령 패널 검증"
description: "Flutter WidgetTester로 스마트홈 MQTT 명령을 ModalBottomSheet에서 선택하는 흐름을 테스트하고, 오버레이·pump 타이밍·드래그 닫기 때문에 실패했던 문제를 정리했다."
date: 2026-09-12
tags: [Flutter, Dart, 테스트, WidgetTester, MQTT, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 MQTT 명령 패널을 테스트하는 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

화면에서 버튼이 보인다고 `ModalBottomSheet` 테스트가 끝나는 건 아니다. Flutter WidgetTester에서는 시트가 오버레이로 올라왔는지, 그 안의 MQTT 명령을 골랐는지, 선택 후 시트가 닫히며 콜백이 한 번 호출됐는지까지 확인해야 한다. 처음에는 `find.text('난방 시작')`만 검사했는데 본문과 시트에 같은 문구가 있어 엉뚱한 위젯을 누르는 테스트가 됐다.

## 테스트 대상은 시트 여는 동작과 분리한다

스마트홈 기기 카드에는 시트를 여는 버튼만 남기고 MQTT 서비스는 콜백 뒤에서 Fake로 격리했다. 실제 브로커 연결을 테스트에 섞으면 네트워크 지연 때문에 시트 동작 자체의 실패 원인을 알기 어렵다.

```dart
class DeviceCommandButton extends StatelessWidget {
  const DeviceCommandButton({super.key, required this.onPublish});

  final Future<void> Function(String command) onPublish;

  @override
  Widget build(BuildContext context) {
    return FilledButton(
      key: const Key('device-command-button'),
      onPressed: () async {
        await showModalBottomSheet<void>(
          context: context,
          builder: (context) => SafeArea(
            key: const Key('mqtt-command-sheet'),
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                ListTile(
                  key: const Key('heat-on-command'),
                  title: const Text('난방 시작'),
                  onTap: () async {
                    await onPublish('heat_on');
                    if (context.mounted) Navigator.pop(context);
                  },
                ),
              ],
            ),
          ),
        );
      },
      child: const Text('기기 명령'),
    );
  }
}
```

`showModalBottomSheet`는 현재 화면의 자식으로 바로 붙지 않고 별도 route와 오버레이를 만든다. 그래서 버튼을 누른 직후 시트의 `ListTile`을 찾으면 실패할 수 있다. 시트를 여는 탭 뒤에 `pump()`를 한 번 넣어 route가 반영될 프레임을 진행한다.

## `tap → pump → 명령 선택 → pump` 순서를 고정한다

아래 테스트는 “시트가 열림”, “정확한 항목을 선택함”, “MQTT payload가 한 번 발행됨”, “시트가 사라짐”을 각각 확인한다. 코드 바로 위에서 필요한 Fake 상태를 준비하는 이유는 네트워크가 아니라 UI 계약만 검증하기 위해서다.

```dart
testWidgets('ModalBottomSheet에서 MQTT 난방 명령을 한 번 발행한다',
    (tester) async {
  final published = <String>[];

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: DeviceCommandButton(
          onPublish: (command) async => published.add(command),
        ),
      ),
    ),
  );

  await tester.tap(find.byKey(const Key('device-command-button')));
  await tester.pump();

  expect(find.byKey(const Key('mqtt-command-sheet')), findsOneWidget);
  expect(find.byKey(const Key('heat-on-command')), findsOneWidget);

  await tester.tap(find.byKey(const Key('heat-on-command')));
  await tester.pump();

  expect(published, ['heat_on']);
  expect(find.byKey(const Key('mqtt-command-sheet')), findsNothing);
});
```

| 검증 지점 | 안정적인 기준 | 흔한 실패 |
| --- | --- | --- |
| 시트 열림 | `pump()` 후 `ModalBottomSheet` | 탭 직후 검색 |
| 명령 선택 | 명령별 `Key` | 중복되는 `find.text()` |
| 발행 횟수 | Fake 리스트 길이 | 화면 문구만 확인 |
| 닫힘 | route가 사라졌는지 확인 | `pumpAndSettle()`에 의존 |

## 드래그 닫기는 별도 테스트로 둔다

사용자가 명령을 고르지 않고 아래로 밀어 닫는 경우도 있다. 이때는 MQTT 발행이 없어야 한다. `pumpAndSettle()`만 사용하면 시트의 전환 애니메이션과 다른 지속 애니메이션이 섞여 테스트가 느려질 수 있어, 필요한 시간만 `pump`한다.

```dart
await tester.dragFrom(const Offset(200, 300), const Offset(200, 700));
await tester.pump(const Duration(milliseconds: 300));

expect(published, isEmpty);
expect(find.byKey(const Key('heat-on-command')), findsNothing);
```

단, 기본 `showModalBottomSheet`가 드래그를 허용하는지는 Flutter 버전과 `enableDrag` 설정에 따라 달라진다. 테스트에서 `enableDrag: false`로 고정해 놓고 드래그 닫힘을 기대하면 당연히 실패한다. 시트의 설정을 확인하고, 닫기 버튼·바깥 영역 탭·드래그를 서로 다른 시나리오로 나누는 편이 읽기도 쉽다.

솔직하게 정리하면, Flutter WidgetTester의 `ModalBottomSheet` 테스트 핵심은 오버레이가 생기는 프레임과 MQTT 명령 발행 시점을 분리하는 일이다. 화면 텍스트 대신 `Key`를 쓰고, Fake 발행 목록과 시트 route의 생존 여부를 함께 검사하면 스마트홈 제어 패널의 중복 발행과 닫힘 오류를 빠르게 잡을 수 있다.
