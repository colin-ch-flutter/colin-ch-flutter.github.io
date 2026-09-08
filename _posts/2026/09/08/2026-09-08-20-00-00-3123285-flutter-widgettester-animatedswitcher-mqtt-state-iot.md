---
layout: post
title: "Flutter WidgetTester AnimatedSwitcher 테스트 - MQTT 연결 상태 전환 검증"
description: "Flutter WidgetTester로 AnimatedSwitcher를 사용하는 MQTT 연결 카드의 로딩·연결·오류 전환을 검증하고, ValueKey와 pump(Duration)으로 애니메이션 테스트가 흔들리지 않게 만드는 방법을 정리했다."
date: 2026-09-08
tags: [Flutter, Dart, MQTT, IoT, 테스트]
comments: true
share: true
---

![Flutter WidgetTester AnimatedSwitcher로 구현한 MQTT 연결 상태 카드](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=1200&q=80)

그림에서 볼 부분은 상태가 바뀔 때 카드 전체를 갈아끼우는 게 아니라, 같은 영역 안에서 로딩·연결·오류 UI가 교체되는 흐름이다.

Flutter WidgetTester에서 `AnimatedSwitcher`를 테스트할 때 `pumpAndSettle()`만 넣으면 끝이라고 생각하기 쉽다. 실제로는 상태별 자식에게 `Key`를 주지 않아 전환이 발생하지 않거나, MQTT 연결 스트림이 계속 살아 있어 테스트가 끝나지 않는 문제가 더 자주 생긴다. `ValueKey`로 상태를 구분하고 필요한 시간만 `pump()`하면 Flutter IoT 연결 카드도 안정적으로 검증할 수 있다.

## 상태 카드에 ValueKey를 붙인다

아래 위젯은 실제 화면의 복잡한 Repository를 빼고, 테스트할 전환 규칙만 남긴 예다. `AnimatedSwitcher`는 같은 타입의 위젯이 연속으로 들어오면 같은 위젯으로 판단할 수 있으므로 상태를 구분하는 키가 필요하다.

```dart
enum MqttStatus { connecting, connected, error }

class MqttStatusCard extends StatelessWidget {
  const MqttStatusCard({super.key, required this.status});

  final MqttStatus status;

  @override
  Widget build(BuildContext context) {
    final child = switch (status) {
      MqttStatus.connecting => const _StatusText(
          key: ValueKey('connecting'),
          text: 'MQTT 연결 중',
        ),
      MqttStatus.connected => const _StatusText(
          key: ValueKey('connected'),
          text: '스마트홈 연결됨',
        ),
      MqttStatus.error => const _StatusText(
          key: ValueKey('error'),
          text: '연결 실패',
        ),
    };

    return AnimatedSwitcher(
      duration: const Duration(milliseconds: 300),
      child: child,
    );
  }
}

class _StatusText extends StatelessWidget {
  const _StatusText({super.key, required this.text});

  final String text;

  @override
  Widget build(BuildContext context) => Text(text);
}
```

`ValueKey`가 없으면 `connecting`에서 `connected`로 바뀌어도 테스트 트리에서 자식이 교체되지 않는 것처럼 보일 수 있다. 처음에는 텍스트만 바뀌면 충분하다고 생각했는데, 실제 카드에는 아이콘·버튼·색상까지 상태별로 달라서 전환 경계가 분명해야 했다.

## pumpAndSettle보다 pump(Duration)를 우선한다

상태를 외부에서 바꿀 수 있는 테스트용 위젯을 만들고, 애니메이션이 진행 중인 프레임과 완료된 프레임을 나눠 확인한다.

```dart
testWidgets('MQTT 연결 상태가 애니메이션으로 전환된다', (tester) async {
  MqttStatus status = MqttStatus.connecting;

  await tester.pumpWidget(
    StatefulBuilder(
      builder: (context, setState) {
        return MaterialApp(
          home: Column(
            children: [
              MqttStatusCard(status: status),
              ElevatedButton(
                onPressed: () => setState(
                  () => status = MqttStatus.connected,
                ),
                child: const Text('연결'),
              ),
            ],
          ),
        );
      },
    ),
  );

  expect(find.text('MQTT 연결 중'), findsOneWidget);

  await tester.tap(find.text('연결'));
  await tester.pump();
  expect(find.text('스마트홈 연결됨'), findsOneWidget);

  await tester.pump(const Duration(milliseconds: 300));
  expect(find.text('MQTT 연결 중'), findsNothing);
  expect(find.text('스마트홈 연결됨'), findsOneWidget);
});
```

`pump()` 직후에는 이전 자식과 새 자식이 잠시 함께 트리에 있을 수 있다. 그래서 전환이 시작됐다는 사실은 새 텍스트의 존재로 확인하고, `duration`만큼 시간을 진행한 뒤 이전 텍스트가 사라졌는지 확인했다. 무조건 `pumpAndSettle()`을 쓰면 MQTT heartbeat나 반복 로딩 애니메이션이 섞인 화면에서 타임아웃이 날 수 있다.

| 검증 대상 | 권장 방식 | 이유 |
| --- | --- | --- |
| 상태 변경 직후 | `pump()` | 전환 시작 프레임 확인 |
| 유한 애니메이션 완료 | `pump(Duration)` | 필요한 시간만 진행 |
| 계속 실행되는 스트림 | 이벤트 주입 후 `pump()` | 화면 정착을 기다리지 않음 |
| 애니메이션이 없는 분기 | `find.text`와 상태 검증 | 불필요한 대기 제거 |

## 테스트에서 놓친 부분

상태만 바꾸고 `AnimatedSwitcher`의 키를 확인하지 않으면 테스트는 통과해도 실제 전환은 일어나지 않을 수 있다. 반대로 키를 매번 랜덤 값으로 만들면 모든 빌드가 새 위젯이 되어 깜빡임이 생긴다. 상태 이름처럼 의미가 고정된 문자열을 키로 쓰는 편이 안전하다.

또 `AnimatedSwitcher` 안에 MQTT 구독을 직접 넣지 않는 게 좋다. 스트림 수명과 화면 애니메이션 수명이 달라서, 위젯이 dispose된 뒤에도 이벤트가 들어오면 테스트가 간헐적으로 실패한다. 연결 상태는 Controller나 Repository가 만들고, 카드는 상태를 받아 표시만 하게 분리했다.

솔직하게 정리하면, Flutter WidgetTester에서 AnimatedSwitcher를 검증하는 핵심은 애니메이션을 빠르게 만드는 게 아니다. 어떤 상태가 이전 자식이고 어떤 상태가 새 자식인지 `ValueKey`로 고정하고, MQTT 스트림의 시간과 UI 전환 시간을 분리하는 데 있다. 이 기준을 잡아두면 연결·재연결·오류 복구 카드도 같은 패턴으로 테스트할 수 있다.
