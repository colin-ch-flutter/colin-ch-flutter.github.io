---
layout: post
title: "Flutter Lottie Widget Test - pumpAndSettle이 멈추는 애니메이션 테스트 해결"
description: "Flutter Lottie 애니메이션이 반복 재생되어 Widget Test의 pumpAndSettle이 끝나지 않는 문제를 AnimationController 주입과 pump(Duration)으로 해결하는 방법을 IoT 화면 예제로 정리했다."
date: 2026-07-21
tags: [Flutter, Dart, Lottie, Riverpod, IoT, 테스트]
comments: true
share: true
---

![Flutter Lottie Widget Test와 IoT 상태 애니메이션](/images/2026-07-21-flutter-lottie-widget-test.png)

그림에서 볼 부분은 스마트홈 화면의 보일러 상태 애니메이션과 아래 테스트 타임라인이다. 애니메이션을 실제 시간에 맡기지 않고 테스트가 시간을 직접 움직이는 구조를 표현했다.

Flutter Lottie Widget Test에서 가장 자주 만난 문제는 `pumpAndSettle()`이 끝나지 않는 현상이었다. 연결 중 아이콘을 계속 반복 재생하도록 `repeat: true`를 켜둔 게 원인이었다. 처음엔 테스트 타임아웃으로 생각했는데, 애니메이션이 계속 프레임을 예약하니 settle 될 상태 자체가 아니었다.

## 문제 상황 — 화면은 맞는데 테스트만 멈춘다

실제 IoT 앱에서는 MQTT 연결 중 로딩 애니메이션을 반복한다.

```dart
Lottie.asset(
  'assets/animations/connecting.json',
  repeat: true,
);
```

이 위젯이 들어간 화면에서 아래처럼 작성하면 테스트가 무한히 기다린다.

```dart
await tester.pumpWidget(const ProviderScope(child: App()));
await tester.pumpAndSettle(); // 반복 애니메이션 때문에 종료되지 않음
```

`pumpAndSettle`은 애니메이션이 끝날 때까지 기다리는 함수다. BLE 스캔이나 MQTT 연결 같은 비동기 작업까지 함께 들어가면 실패 원인을 로그만 보고 찾기 더 어렵다.

## 해결 기준 — 애니메이션을 끄는 게 아니라 시간을 주입한다

운영 화면과 테스트 화면을 `bool isTest`로 나누는 방법도 있지만, 조건문이 제품 코드 전체로 퍼진다. 내가 사용한 방식은 `AnimationController`를 선택적으로 주입하는 것이다. 기본값은 운영용 반복 재생이고, 테스트에서는 컨트롤러를 멈춘 상태로 넣는다.

```dart
class ConnectionAnimation extends StatefulWidget {
  const ConnectionAnimation({
    required this.isConnected,
    this.testController,
    super.key,
  });

  final bool isConnected;
  final AnimationController? testController;

  @override
  State<ConnectionAnimation> createState() => _ConnectionAnimationState();
}

class _ConnectionAnimationState extends State<ConnectionAnimation>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller =
      widget.testController ?? AnimationController(
        vsync: this,
        duration: const Duration(milliseconds: 900),
      )..repeat();

  @override
  Widget build(BuildContext context) {
    if (widget.isConnected) {
      return const Icon(Icons.check_circle, key: Key('connected'));
    }

    return Lottie.asset(
      'assets/animations/connecting.json',
      controller: _controller,
      key: const Key('connecting'),
      repeat: widget.testController == null,
    );
  }

  @override
  void dispose() {
    if (widget.testController == null) _controller.dispose();
    super.dispose();
  }
}
```

핵심은 외부에서 받은 컨트롤러를 위젯이 dispose하지 않는다는 점이다. 소유권까지 빼앗으면 테스트 tearDown에서 이중 dispose가 발생한다. 실제 프로젝트에서도 이 부분을 놓쳐 `AnimationController.dispose() called more than once` 오류를 한 번 봤다.

## Widget Test — pump으로 상태 전이를 확인한다

테스트에서는 `TestVSync`와 유한한 컨트롤러를 만들어 특정 시점의 화면을 검증한다.

```dart
testWidgets('연결되면 Lottie 대신 완료 아이콘을 보여준다', (tester) async {
  final controller = AnimationController(
    vsync: const TestVSync(),
    duration: const Duration(milliseconds: 900),
  );
  addTearDown(controller.dispose);

  await tester.pumpWidget(
    MaterialApp(
      home: ConnectionAnimation(
        isConnected: false,
        testController: controller,
      ),
    ),
  );
  expect(find.byKey(const Key('connecting')), findsOneWidget);

  controller.value = 0.5;
  await tester.pump();
  expect(controller.value, closeTo(0.5, 0.001));

  await tester.pumpWidget(
    const MaterialApp(home: ConnectionAnimation(isConnected: true)),
  );
  await tester.pump();
  expect(find.byKey(const Key('connected')), findsOneWidget);
});
```

| 방식 | 적합한 상황 | 주의점 |
|---|---|---|
| `pumpAndSettle()` | 종료되는 짧은 전환 | 반복 Lottie에서는 사용 금지 |
| `pump(Duration)` | 특정 시간 뒤 상태 | duration을 고정해야 재현 가능 |
| 컨트롤러 주입 | 애니메이션 상태 자체 검증 | dispose 소유권을 명확히 구분 |

## 짧은 요점 정리

Flutter Lottie Widget Test가 멈추면 타임아웃 시간을 늘리기 전에 `repeat` 애니메이션부터 의심해야 한다. 테스트용 플래그를 여러 곳에 뿌리는 대신 컨트롤러를 주입하고, `pump(Duration)` 또는 `controller.value`로 원하는 프레임을 지정하면 BLE·MQTT 연결 화면도 안정적으로 검증할 수 있다. `pumpAndSettle()`은 애니메이션이 정말 끝나는 화면에서만 쓰는 게 안전하다.
