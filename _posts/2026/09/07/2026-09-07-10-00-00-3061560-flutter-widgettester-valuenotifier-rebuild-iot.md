---
layout: post
title: "Flutter WidgetTester ValueNotifier 테스트 - 스마트홈 카드 불필요한 리빌드 잡기"
description: "Flutter WidgetTester에서 ValueNotifier와 ValueListenableBuilder로 스마트홈 카드의 상태 변경과 불필요한 리빌드를 검증하는 방법을 정리했다."
date: 2026-09-07
tags: [Flutter, Dart, 테스트, WidgetTester, 성능최적화, IoT, 스마트홈]
comments: true
share: true
---
![Flutter WidgetTester와 ValueNotifier로 스마트홈 카드 리빌드 테스트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

_이 그림에서 볼 부분은 한 카드의 상태 변경이 전체 스마트홈 화면으로 번지지 않도록 테스트 경계를 나눈 구조다._

Flutter WidgetTester에서 화면이 제대로 보이는지만 확인하면 불필요한 리빌드를 놓치기 쉽다. 스마트홈 대시보드처럼 MQTT 온도 값이 자주 바뀌는 화면에서는 `ValueNotifier`와 `ValueListenableBuilder`를 작은 테스트 경계로 만들고, 필요한 카드만 다시 빌드되는지 검증하는 편이 안전하다.

## 전체 화면을 감시하면 원인을 놓친다

처음에는 테스트에서 `find.text('22.5°C')`만 확인했다. 해보니 온도 값이 바뀔 때 보일러 전원 카드와 연결 상태 배너까지 같이 `build()`를 타고 있었다. 화면은 맞게 보이지만, 기기가 10대가 되면 MQTT 이벤트 하나가 전체 대시보드를 흔든다.

| 상태 보관 방식 | 변경 시 영향 | 테스트하기 좋은 범위 |
|---|---|---|
| 화면 전체 `setState` | 자식 위젯 전부 리빌드 | 단순한 정적 화면 |
| `ChangeNotifier` 하나 | listener 전체 호출 | 여러 필드가 함께 바뀌는 폼 |
| 카드별 `ValueNotifier` | 해당 `ValueListenableBuilder`만 갱신 | 온도·전원처럼 독립적인 IoT 값 |

`ValueNotifier`는 값의 동일성 비교가 단순하다는 제한이 있지만, 카드 단위 상태를 작게 나누는 데는 오히려 장점이 있다. 테스트에서 “이 값이 바뀌면 이 카드만 움직인다”를 분명히 말할 수 있기 때문이다.

## 리빌드 횟수를 기록하는 테스트 위젯

실제 카드 대신 리빌드 횟수를 기록하는 작은 위젯을 만들어 상태 경계를 먼저 검증한다.

```dart
class RebuildProbe extends StatelessWidget {
  const RebuildProbe({
    required this.name,
    required this.child,
    required this.onBuild,
    super.key,
  });

  final String name;
  final Widget child;
  final void Function(String name) onBuild;

  @override
  Widget build(BuildContext context) {
    onBuild(name);
    return child;
  }
}

void _recordBuild(String name) {
  // 테스트에서는 name별 카운터를 올린다.
}

class BoilerDashboard extends StatelessWidget {
  const BoilerDashboard({required this.temperature, super.key});

  final ValueNotifier<double> temperature;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ValueListenableBuilder<double>(
          valueListenable: temperature,
          builder: (_, value, __) => Text('${value.toStringAsFixed(1)}°C'),
        ),
        const RebuildProbe(
          name: 'power-card',
          onBuild: _recordBuild,
          child: Text('보일러 전원'),
        ),
      ],
    );
  }
}
```

`RebuildProbe`의 `_recordBuild`는 예제에서는 전역 카운터나 테스트용 콜백으로 연결한다. 운영 코드에 디버그 카운터를 넣기보다, 실제 프로젝트에서는 카드 내부를 별도 위젯으로 분리한 뒤 테스트 전용 래퍼를 두는 편이 덜 지저분하다.

## `ValueNotifier` 변경 뒤 필요한 만큼만 pump한다

값을 바꾼 직후 바로 assertion하면 위젯 트리가 아직 갱신되지 않았을 수 있다. 이때 전체 애니메이션을 기다리는 `pumpAndSettle()`보다 한 프레임만 진행하는 `pump()`이 의도를 정확히 표현한다.

```dart
testWidgets('온도 변경은 온도 카드만 갱신한다', (tester) async {
  final temperature = ValueNotifier<double>(21.0);
  addTearDown(temperature.dispose);

  await tester.pumpWidget(
    MaterialApp(home: BoilerDashboard(temperature: temperature)),
  );

  expect(find.text('21.0°C'), findsOneWidget);
  expect(find.text('보일러 전원'), findsOneWidget);

  temperature.value = 22.5;
  await tester.pump();

  expect(find.text('22.5°C'), findsOneWidget);
  expect(find.text('21.0°C'), findsNothing);
});
```

여기서 테스트가 증명하는 것은 “값이 바뀌었다”뿐이다. 실제 리빌드 범위까지 확인하려면 카드별 build counter를 넣거나 `debugPrintRebuildDirtyWidgets` 로그를 별도 프로파일링 테스트에서 확인해야 한다. 위젯 테스트 assertion에 내부 구현 횟수를 과하게 고정하면 레이아웃 리팩터링 때 테스트가 쉽게 깨진다.

## 실제로 헷갈렸던 지점

처음에는 `ValueNotifier<Map<String, double>>` 하나에 모든 기기 온도를 넣었다. Map 내부 값만 바꾸고 `notifyListeners()`를 직접 호출하는 코드가 생겼고, 어느 카드가 영향을 받는지 추적하기 어려웠다. 카드별 notifier로 쪼개자 테스트 준비 코드는 조금 늘었지만, MQTT 이벤트와 UI 변경의 경계가 보였다.

다만 `ValueNotifier<List<Device>>`처럼 mutable 객체를 그대로 넣는 방식은 피해야 한다. 리스트 내부만 수정하면 notifier의 `value` 참조가 같아서 알림이 발생하지 않을 수 있다. 새 리스트를 대입하거나, 컬렉션 변경이 핵심이면 `ChangeNotifier`의 명시적 메서드로 상태 변경을 감싸는 편이 낫다.

최근 [Flutter WidgetTester에서 MQTT 스트림을 `pump()`으로 제어한 글]({% post_url 2026-09-06-20-00-00-3049215-flutter-widgettester-pumpandsettle-mqtt-stream-iot %})과 같이, 계속 흐르는 IoT 데이터와 화면 프레임을 분리해야 테스트가 흔들리지 않는다.

짧게 정리하면 이렇다.

- 독립적인 IoT 카드 상태는 `ValueNotifier`로 작게 나눈다.
- 값 변경 뒤에는 필요한 프레임만 `pump()`한다.
- 리빌드 횟수 자체보다 화면 경계와 결과를 우선 assertion한다.
- notifier와 stream은 `addTearDown()`에서 반드시 정리한다.
