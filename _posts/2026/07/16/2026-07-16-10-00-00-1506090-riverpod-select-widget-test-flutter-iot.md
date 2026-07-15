---
layout: post
title: "Flutter IoT Riverpod select 테스트 - MQTT 상태 변경 때 불필요한 Widget rebuild 막기"
description: "Flutter IoT 스마트홈에서 Riverpod select로 MQTT 온도 변경과 BLE 연결 상태를 분리하고, 불필요한 Widget rebuild를 테스트하는 방법을 정리했다."
date: 2026-07-16
tags: [Flutter, Dart, Riverpod, MQTT, BLE, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod select Widget Test 스마트홈 기기 카드](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

Flutter IoT 스마트홈 화면에서 MQTT 온도 메시지가 들어올 때마다 BLE 연결 아이콘까지 다시 그려지는 문제가 있었다. Riverpod `select`로 카드가 실제로 사용하는 값만 구독하고, Widget Test에서 rebuild 횟수를 확인하니 불필요한 렌더링을 줄일 수 있었다.

## 상태 전체를 watch하면 카드가 같이 흔들린다

보일러 카드가 아래처럼 여러 값을 가진다고 하자.

| 상태 | 변경 빈도 | 사용하는 화면 |
| --- | ---: | --- |
| 온도 | MQTT 응답마다 | 온도 텍스트 |
| 전원 | 명령 응답 때 | 토글 |
| BLE 연결 | 재연결 때 | 연결 아이콘 |
| 마지막 수신 시각 | 메시지마다 | 디버그 라벨 |

카드 전체에서 `ref.watch(deviceStateProvider(deviceId))`를 호출하면 온도만 바뀌어도 전원 토글과 연결 아이콘이 다시 build된다. 처음엔 이 정도는 괜찮다고 생각했는데, 기기 12개가 동시에 메시지를 받으니 스크롤 중 프레임 드롭과 로그 중복이 눈에 띄었다.

## select로 필요한 필드만 구독한다

온도 위젯은 `DeviceState` 전체가 아니라 `temperature`만 읽도록 잘랐다.

```dart
class TemperatureText extends ConsumerWidget {
  const TemperatureText({required this.deviceId, super.key});

  final String deviceId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final temperature = ref.watch(
      deviceStateProvider(deviceId).select((state) => state.temperature),
    );

    return Text('${temperature.toStringAsFixed(1)}°C');
  }
}
```

`select` 안에서 새 객체를 매번 만들면 비교 기준이 무너진다. 문자열 포맷이나 `copyWith` 결과를 select 안에서 만들지 않고, 원시값이나 동등성 비교가 확실한 값만 반환하는 편이 안전했다. 여러 필드가 꼭 필요하면 별도 상태 모델을 만들고 `==`와 `hashCode`도 같이 정의해야 한다.

## Widget Test에서 rebuild 횟수를 센다

화면이 빨라졌다는 느낌만으로 끝내지 않고, 온도 변경에 연결 아이콘이 다시 build되지 않는지 카운터로 확인했다.

```dart
testWidgets('온도 변경은 연결 아이콘을 rebuild하지 않는다', (tester) async {
  final fake = FakeDeviceRepository(DeviceState.connected(
    deviceId: 'boiler-1',
    temperature: 21.0,
  ));

  await tester.pumpWidget(
    ProviderScope(
      overrides: [deviceRepositoryProvider.overrideWithValue(fake)],
      child: const MaterialApp(home: BoilerCard(deviceId: 'boiler-1')),
    ),
  );

  expect(find.text('21.0°C'), findsOneWidget);
  expect(find.byKey(const Key('ble-status')), findsOneWidget);
  expect(fake.connectionWidgetBuilds, 1);

  fake.emit(DeviceState.connected(
    deviceId: 'boiler-1',
    temperature: 22.0,
  ));
  await tester.pump();

  expect(find.text('22.0°C'), findsOneWidget);
  expect(fake.connectionWidgetBuilds, 1);
});
```

실제 프로젝트에서는 `connectionWidgetBuilds`를 Repository가 아니라 테스트 전용 `BleStatusBadge`의 `buildCount`로 두는 편이 더 정확하다. Repository 호출 횟수와 Widget rebuild 횟수는 다른 지표이기 때문이다. 이걸 섞어서 검증했더니 데이터는 한 번만 읽는데 화면은 두 번 그려지는 원인을 놓쳤다.

## 주의할 경계

`select`는 네트워크 메시지를 줄이는 기능이 아니다. MQTT 구독과 Provider가 받는 이벤트 수는 그대로고, 변경된 값이 선택 결과와 같을 때 하위 위젯 rebuild를 막는 역할만 한다. `select`를 붙였는데도 화면 전체가 다시 그려진다면 부모가 상태 전체를 watch하고 있거나, `Consumer` 범위가 너무 큰지 확인해야 한다.

또 `double` 값은 센서 노이즈로 21.01, 21.02처럼 계속 달라질 수 있다. 표시 단위가 0.1도라면 Provider에서 반올림한 값을 제공하거나, UI 모델에서 비교 기준을 정해야 한다. 그렇지 않으면 사용자가 보기에 같은 `21.0°C`인데도 rebuild가 계속 발생한다.

짧게 정리하면:

- 기기 상태 전체가 아니라 위젯이 쓰는 필드만 `select`한다.
- `select` 안에서 매번 새 객체를 만들지 않는다.
- Widget Test에서는 Repository 호출과 rebuild 횟수를 분리해 검증한다.
- 센서 노이즈와 부모 위젯의 큰 watch 범위를 함께 확인한다.
