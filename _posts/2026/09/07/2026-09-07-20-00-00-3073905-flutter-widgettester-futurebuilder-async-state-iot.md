---
layout: post
title: "Flutter WidgetTester FutureBuilder 테스트 - 스마트홈 로딩·에러 상태 검증"
description: "Flutter WidgetTester에서 FutureBuilder의 로딩·성공·에러 상태를 pump 순서로 검증하고, 스마트홈 대시보드 테스트가 무한 대기하지 않게 만드는 방법을 정리했다."
date: 2026-09-07
tags: [Flutter, Dart, 테스트, WidgetTester, IoT, 스마트홈]
comments: true
share: true
---
![Flutter WidgetTester FutureBuilder로 스마트홈 비동기 상태 테스트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

_이 그림에서 볼 부분은 스마트홈 카드가 데이터를 기다리는 동안에도 로딩과 에러를 별도 상태로 보여주는 흐름이다._

Flutter WidgetTester에서 `FutureBuilder`를 테스트할 때는 `pumpAndSettle()` 하나로 끝내려 하면 안 된다. 스마트홈 대시보드의 API와 MQTT 초기화가 함께 묶여 있으면 Future가 끝나지 않는 순간 테스트도 같이 멈춘다. `Completer`로 응답 시점을 직접 제어하고, `pump()`를 상태 변화 사이에 나눠 호출하는 방식이 훨씬 안정적이다.

## 화면이 맞는데도 테스트가 불안정했던 이유

처음에는 위젯을 띄운 뒤 `pumpAndSettle()`을 호출하고 온도 텍스트를 찾았다. 로컬에서는 통과했다. 그런데 CI에서 간헐적으로 타임아웃이 났다. 실제 화면에는 MQTT 연결을 기다리는 Future와 주기적인 상태 갱신 Timer가 같이 있었고, settle 조건이 영원히 만족되지 않을 수 있었다.

이번 테스트에서 확인할 상태는 세 가지다.

| 상태 | 화면에서 보여줄 것 | 테스트 시점 |
|---|---|---|
| 로딩 | `기기 정보를 불러오는 중` | 첫 `pump()` 직후 |
| 성공 | 기기 이름과 현재 온도 | Future 완료 후 `pump()` |
| 실패 | 재시도 가능한 오류 문구 | Future에 예외 전달 후 `pump()` |

## 테스트 대상 위젯을 작게 만든다

Repository를 직접 호출하지 않고 Future를 주입할 수 있게 하면 네트워크와 테스트 타이밍이 분리된다. 다음 코드는 실제 IoT 화면에서 필요한 상태 전환만 남긴 예시다.

```dart
class DeviceSummary {
  const DeviceSummary({required this.name, required this.temperature});

  final String name;
  final double temperature;
}

class DeviceCard extends StatelessWidget {
  const DeviceCard({required this.loadSummary, super.key});

  final Future<DeviceSummary> Function() loadSummary;

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<DeviceSummary>(
      future: loadSummary(),
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return const Text('기기 정보를 불러오는 중');
        }
        if (snapshot.hasError) {
          return const Text('기기 정보를 불러오지 못했다');
        }

        final summary = snapshot.requireData;
        return Text('${summary.name} · ${summary.temperature}°C');
      },
    );
  }
}
```

`future`를 `build()` 안에서 매번 새로 만들면 부모 위젯이 리빌드될 때 요청도 다시 시작된다. 실제 화면에서는 Repository에서 Future를 한 번 만든 뒤 보관하는 편이 맞다. 여기서는 WidgetTester가 Future 완료 전후를 제어하는 데 초점을 둔다.

## Completer로 로딩과 성공을 나눠 검증한다

아래 테스트처럼 Future를 즉시 완료시키지 않으면 로딩 UI가 실제로 렌더링됐는지 확인할 수 있다.

```dart
testWidgets('로딩 후 기기 요약을 보여준다', (tester) async {
  final completer = Completer<DeviceSummary>();

  await tester.pumpWidget(
    MaterialApp(
      home: DeviceCard(loadSummary: () => completer.future),
    ),
  );

  expect(find.text('기기 정보를 불러오는 중'), findsOneWidget);

  completer.complete(
    const DeviceSummary(name: '거실 보일러', temperature: 22.5),
  );
  await tester.pump();

  expect(find.text('거실 보일러 · 22.5°C'), findsOneWidget);
  expect(find.text('기기 정보를 불러오는 중'), findsNothing);
});
```

여기서 `pump()`는 단순히 화면을 다시 그리는 호출이 아니다. 완료된 Future의 마이크로태스크를 처리하고 `FutureBuilder`가 새 snapshot을 받는 시점까지 테스트 프레임을 진행한다. `complete()`만 호출하고 `pump()`를 빼먹으면 데이터는 준비됐는데 화면은 여전히 로딩 상태처럼 보인다.

## 에러는 예외 타입보다 사용자 상태를 확인한다

MQTT 연결 실패의 실제 예외 메시지는 플랫폼과 라이브러리 버전에 따라 달라질 수 있다. Widget 테스트에서는 내부 메시지 전체보다 사용자가 다시 시도할 수 있는 상태가 나왔는지를 검증하는 쪽이 덜 깨진다.

```dart
testWidgets('기기 조회 실패 시 에러 문구를 보여준다', (tester) async {
  final completer = Completer<DeviceSummary>();

  await tester.pumpWidget(
    MaterialApp(
      home: DeviceCard(loadSummary: () => completer.future),
    ),
  );

  completer.completeError(StateError('MQTT broker unavailable'));
  await tester.pump();

  expect(find.text('기기 정보를 불러오지 못했다'), findsOneWidget);
  expect(find.text('MQTT broker unavailable'), findsNothing);
});
```

주의할 점은 `FutureBuilder`의 Future를 `build()` 안에서 생성하는 패턴이다. 이 상태에서 부모가 1초마다 MQTT 값을 반영하면 테스트가 아니라 실제 앱에서도 요청이 반복될 수 있다. 로딩 테스트가 자꾸 재현되지 않는다면 `pump` 횟수보다 Future 생성 위치부터 확인해야 한다.

짧게 정리하면, 첫 `pump()`에서는 로딩을 확인하고, `Completer`를 완료한 뒤 두 번째 `pump()`에서 성공 또는 실패 상태를 확인하면 된다. 계속 도는 Timer나 MQTT 스트림이 있는 화면은 `pumpAndSettle()`보다 명시적인 `pump()`가 안전하다.
