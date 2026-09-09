---
layout: post
title: "Flutter WidgetTester 제스처 테스트 - 스마트홈 카드 탭과 슬라이더 검증"
description: "Flutter WidgetTester로 스마트홈 제어 카드의 탭·드래그 제스처를 테스트하고, onTap 콜백과 Slider 값 변경이 MQTT 명령으로 이어지는지 안정적으로 검증하는 방법을 정리했다."
date: 2026-09-10
tags: [Flutter, Dart, 테스트, WidgetTester, IoT, MQTT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 제어 카드 제스처를 테스트하는 화면](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

그림에서 봐야 할 부분은 화면을 누르는 동작과 테스트 체크포인트가 한 흐름으로 연결된 점이다.

Flutter WidgetTester에서 `tap()`만 호출하면 버튼 테스트는 끝난 것처럼 보인다. 그런데 스마트홈 화면의 제어 카드는 `GestureDetector`로 감싼 뒤 내부에 `Slider`를 넣는 경우가 많다. 탭은 동작하는데 슬라이더 드래그가 카드의 콜백까지 건드리거나, MQTT 명령을 두 번 보내는 문제가 생겼다. 이번에는 UI를 실제로 움직인 뒤 Controller의 상태와 Repository 호출을 함께 확인했다.

## 탭과 드래그는 다른 계약이다

테스트에서 확인할 동작을 이렇게 나눴다.

| 사용자 동작 | 검증할 결과 | 흔한 실패 |
| --- | --- | --- |
| 카드 탭 | 선택된 기기로 이동하거나 전원 명령 1회 호출 | `GestureDetector` 중첩으로 중복 호출 |
| 스위치 탭 | `isOn` 상태 변경 | `pump()` 누락으로 이전 화면 검사 |
| 온도 슬라이더 드래그 | 목표 온도 변경 및 명령 1회 호출 | 좌표가 화면 밖이라 드래그 실패 |
| 빠른 연속 드래그 | 마지막 값만 전송하거나 debounce | 중간 값까지 MQTT 발행 |

테스트 대상은 실제 MQTT Client가 아니라 이미 분리해 둔 Fake Repository다. 네트워크를 붙이면 테스트가 느려지고, 연결 상태에 따라 결과가 흔들린다.

## 테스트용 제어 카드를 단순하게 만든다

콜백과 값을 외부에서 주입할 수 있는 카드로 만든다. 이렇게 해야 위젯 테스트가 MQTT 구현 세부사항을 알 필요가 없다.

```dart
class BoilerCard extends StatelessWidget {
  const BoilerCard({
    required this.isOn,
    required this.targetTemperature,
    required this.onCardTap,
    required this.onPowerChanged,
    required this.onTemperatureChanged,
    super.key,
  });

  final bool isOn;
  final double targetTemperature;
  final VoidCallback onCardTap;
  final ValueChanged<bool> onPowerChanged;
  final ValueChanged<double> onTemperatureChanged;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      key: const Key('boiler-card'),
      onTap: onCardTap,
      child: Card(
        child: Column(
          children: [
            Text(isOn ? '가동 중' : '꺼짐'),
            Switch(
              key: const Key('power-switch'),
              value: isOn,
              onChanged: onPowerChanged,
            ),
            Slider(
              key: const Key('temperature-slider'),
              min: 15,
              max: 30,
              value: targetTemperature,
              onChanged: onTemperatureChanged,
            ),
          ],
        ),
      ),
    );
  }
}
```

카드 전체의 `onTap`과 자식 `Switch`, `Slider`는 서로 다른 사용자 의도다. 실제 앱에서 자식 제스처가 부모 탭까지 전파되는 것처럼 보였다면 위젯을 탓하기 전에 콜백에서 같은 명령을 중복 처리하고 있지 않은지 확인해야 한다.

## `tap()`은 Finder를 좁혀서 쓴다

Controller가 상태를 바꾸고 Repository에 한 번만 명령을 보내는 구조를 Fake로 감싼 테스트다.

```dart
testWidgets('전원 스위치를 탭하면 MQTT 전원 명령을 한 번 보낸다',
    (tester) async {
  final repository = FakeDeviceRepository();
  final controller = BoilerController(repository);

  await tester.pumpWidget(
    MaterialApp(
      home: Obx(() => BoilerCard(
            isOn: controller.isOn.value,
            targetTemperature: controller.targetTemperature.value,
            onCardTap: controller.openDetail,
            onPowerChanged: controller.setPower,
            onTemperatureChanged: controller.setTemperature,
          )),
    ),
  );

  await tester.tap(find.byKey(const Key('power-switch')));
  await tester.pump();

  expect(controller.isOn.value, isTrue);
  expect(repository.powerCommands, [true]);
  expect(find.text('가동 중'), findsOneWidget);
});
```

처음에는 `find.byType(Switch)`를 사용했다. 화면에 방별 스위치가 두 개 추가되자 엉뚱한 카드가 탭됐다. 테스트 전용이더라도 제어 의도가 드러나는 `Key`를 부여하는 편이 유지보수하기 쉽다. `tap()` 직후에는 `pump()`을 호출해야 반응형 상태 변경이 반영된 위젯을 검사할 수 있다.

## 슬라이더는 좌표보다 Finder 기준으로 드래그한다

슬라이더는 `drag()`의 이동 거리가 값으로 환산된다. 화면 전체 좌표를 하드코딩하면 기기 폭이 달라질 때 바로 깨진다.

```dart
testWidgets('온도 슬라이더 드래그로 목표 온도를 바꾼다', (tester) async {
  final repository = FakeDeviceRepository();
  final controller = BoilerController(repository);

  await tester.pumpWidget(buildBoilerTestApp(controller));

  final slider = find.byKey(const Key('temperature-slider'));
  expect(tester.widget<Slider>(slider).value, 20);

  await tester.drag(slider, const Offset(90, 0));
  await tester.pump();

  expect(controller.targetTemperature.value, greaterThan(20));
  expect(repository.temperatureCommands.last, greaterThan(20));
});
```

이 테스트에서 이동 거리 `90`은 픽셀 단위라서 기기별로 정확히 같은 온도를 기대하지 않았다. 중요한 것은 범위 안에서 값이 변했는지와 Repository 호출이 발생했는지다. 특정 값이 꼭 필요하면 `Slider` 자체보다 Controller의 `setTemperature(24)`를 단위 테스트로 분리하고, WidgetTester에서는 드래그가 콜백까지 도달하는지만 확인하는 편이 안정적이다.

## debounce가 있으면 `pump` 시간을 명시한다

온도마다 MQTT를 발행하지 않도록 `Timer`로 debounce를 넣었다면 `pump()`만으로는 부족하다. 타이머가 끝나는 시간을 직접 흘려보낸다.

```dart
await tester.drag(slider, const Offset(90, 0));
await tester.pump();
expect(repository.temperatureCommands, isEmpty);

await tester.pump(const Duration(milliseconds: 350));
expect(repository.temperatureCommands, hasLength(1));
```

반대로 `pumpAndSettle()`은 반복 애니메이션이나 연결 상태 Stream이 계속 살아 있는 화면에서 끝나지 않을 수 있다. 내 테스트도 MQTT 연결 아이콘에 무한 애니메이션을 넣은 뒤 여기서 멈췄다. 제스처 직후 한 프레임, debounce 뒤 350ms처럼 필요한 시간만 명시하니 원인 추적이 쉬워졌다.

## 짧게 정리

- 카드 탭, 스위치 탭, 슬라이더 드래그를 서로 다른 계약으로 테스트한다.
- Finder에는 `Key`를 사용하고, 제스처 뒤 상태 반영을 위해 필요한 만큼만 `pump()`한다.
- 드래그 좌표로 특정 숫자를 과하게 보장하지 말고, 정확한 값 계산은 Controller 단위 테스트로 분리한다.
- debounce·애니메이션이 있으면 `pumpAndSettle()` 대신 시간을 직접 제어한다.

스마트홈 UI의 제스처 테스트는 화면이 움직였다는 확인에서 끝나지 않는다. 최종 상태와 MQTT 명령 횟수까지 같이 검사해야 실제 사용자 동작에서 생기는 중복 발행을 잡을 수 있다.
