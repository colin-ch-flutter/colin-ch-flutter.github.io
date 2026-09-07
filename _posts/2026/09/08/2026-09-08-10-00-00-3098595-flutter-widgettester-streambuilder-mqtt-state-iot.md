---
layout: post
title: "Flutter WidgetTester StreamBuilder 테스트 - MQTT 연결 상태 전환 검증"
description: "Flutter WidgetTester로 StreamBuilder 기반 스마트홈 화면의 MQTT 연결 중·연결됨·끊김 상태를 재현하고, pump 순서와 StreamController 정리까지 검증하는 방법을 정리했다."
date: 2026-09-08
tags: [Flutter, Dart, 테스트, WidgetTester, StreamBuilder, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester로 StreamBuilder MQTT 상태를 테스트하는 스마트홈 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter WidgetTester에서 `StreamBuilder` 화면을 테스트할 때는 스트림에 값을 넣는 것만으로 부족하다. 스마트홈 대시보드의 MQTT 연결 상태는 `연결 중 → 연결됨 → 끊김`처럼 순서가 있고, 각 이벤트가 화면에 반영될 때마다 프레임을 진행해야 한다. 이번 테스트에서는 실제 MQTT 브로커 대신 `StreamController`를 주입해 이 상태 전환을 짧고 재현 가능하게 만들었다.

## 화면보다 상태 순서를 검증해야 했다

처음에는 `pumpAndSettle()`을 호출하고 마지막 텍스트만 확인했다. 해보니 테스트는 통과했지만 연결 중 화면이 한 번이라도 보였는지는 알 수 없었다. 반대로 실제 MQTT 스트림은 계속 살아 있으므로 `pumpAndSettle()`이 끝나지 않는 경우도 있었다.

상태별로 필요한 검증은 아래처럼 나뉜다.

| 스트림 이벤트 | 화면 표시 | 테스트 포인트 |
|---|---|---|
| `connecting` | MQTT 연결 중 | 로딩 표시와 안내 문구 |
| `connected` | 온라인 | 제어 버튼 활성화 |
| `disconnected` | 연결 끊김 | 재연결 버튼 노출 |

그림에서 볼 부분은 하나다. 스트림 이벤트가 들어오는 순간과 화면 프레임이 갱신되는 순간은 서로 다르다.

## 테스트 대상 위젯과 Fake 스트림

실제 앱에서는 Repository가 MQTT 상태를 제공하지만, 위젯은 `Stream<MqttConnectionState>`만 알면 된다. 테스트에서는 이 경계를 `StreamController`로 만든다.

```dart
enum MqttConnectionState { connecting, connected, disconnected }

class ConnectionBanner extends StatelessWidget {
  const ConnectionBanner({required this.states, super.key});

  final Stream<MqttConnectionState> states;

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<MqttConnectionState>(
      stream: states,
      initialData: MqttConnectionState.connecting,
      builder: (context, snapshot) {
        return switch (snapshot.data) {
          MqttConnectionState.connected => const Column(
              children: [
                Text('온라인'),
                ElevatedButton(onPressed: null, child: Text('보일러 제어')),
              ],
            ),
          MqttConnectionState.disconnected => Column(
              children: [
                const Text('연결 끊김'),
                ElevatedButton(onPressed: () {}, child: Text('재연결')),
              ],
            ),
          _ => const Column(
              children: [CircularProgressIndicator(), Text('MQTT 연결 중')],
            ),
        };
      },
    );
  }
}
```

`initialData`를 넣어 둔 이유는 첫 프레임에서 빈 화면을 만들지 않기 위해서다. 실제 앱에서도 MQTT 연결을 시작하는 순간에는 중립 상태나 로딩 상태가 있는 편이 사용자 경험과 테스트 양쪽에 유리하다.

## pump 사이에 이벤트를 넣는다

이제 이벤트를 발행하고 그 사이에 `pump()`을 넣는다. 이벤트를 넣은 직후 `find.text()`를 호출하면 아직 빌드가 끝나지 않아 이전 상태를 읽을 수 있다.

```dart
testWidgets('MQTT 연결 상태 전환을 순서대로 표시한다', (tester) async {
  final controller = StreamController<MqttConnectionState>();

  await tester.pumpWidget(
    MaterialApp(
      home: Scaffold(
        body: ConnectionBanner(states: controller.stream),
      ),
    ),
  );

  expect(find.text('MQTT 연결 중'), findsOneWidget);
  expect(find.byType(CircularProgressIndicator), findsOneWidget);

  controller.add(MqttConnectionState.connected);
  await tester.pump();

  expect(find.text('온라인'), findsOneWidget);
  expect(find.text('보일러 제어'), findsOneWidget);
  expect(find.text('MQTT 연결 중'), findsNothing);

  controller.add(MqttConnectionState.disconnected);
  await tester.pump();

  expect(find.text('연결 끊김'), findsOneWidget);
  expect(find.text('재연결'), findsOneWidget);

  await controller.close();
});
```

여기서 `pump()`은 시간을 오래 기다리는 함수가 아니다. 위젯 트리가 스트림 이벤트를 처리하고 한 번 다시 빌드될 기회를 주는 호출이다. 애니메이션 시간이나 MQTT 재연결 타이머까지 함께 검증해야 하는 상황이 아니라면 `pump(const Duration(seconds: 1))`보다 인자 없는 `pump()`이 의도를 더 정확히 표현한다.

## 실제로 막혔던 지점

테스트가 간헐적으로 실패했을 때 `controller.add()`가 유실된 줄 알고 스트림을 여러 번 만들었다. 원인은 테스트 종료 후에도 `StreamController`가 열려 있던 것이었다. 테스트마다 컨트롤러를 새로 만들고 `close()`까지 호출하니 열린 스트림 경고와 다음 테스트에 남는 이벤트가 사라졌다.

상태 스트림 테스트에서 특히 피할 패턴은 세 가지다.

- 계속 이벤트를 내보내는 실제 MQTT 클라이언트를 테스트에 직접 연결한다.
- 모든 상태 변화를 `pumpAndSettle()` 한 번으로 기다린다.
- `StreamController`를 닫지 않고 테스트를 끝낸다.

솔직하게 정리하면 `StreamBuilder` 자체보다 이벤트와 프레임의 경계를 분리하는 일이 핵심이었다. `StreamController`로 상태를 주입하고, 이벤트 하나마다 `pump()`한 뒤, 화면 문구와 버튼을 상태별로 확인하면 MQTT 연결 상태 테스트가 훨씬 덜 흔들린다.
