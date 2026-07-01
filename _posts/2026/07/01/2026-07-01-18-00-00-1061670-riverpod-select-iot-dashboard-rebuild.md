---
layout: post
title: "Flutter IoT Riverpod select - 스마트홈 대시보드 리빌드 줄이기"
description: "Flutter IoT 앱의 Flutter 스마트홈 대시보드에서 Riverpod select로 Flutter BLE와 MQTT 상태 변경 리빌드를 줄인 기준을 정리했다."
date: 2026-07-01
tags: [Flutter, Dart, BLE, MQTT, IoT]
comments: true
share: true
---
![Flutter IoT Riverpod select](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

Flutter IoT 앱에서 Flutter 스마트홈 대시보드를 Riverpod으로 옮길 때 `select`는 꽤 현실적인 최적화였다. Flutter BLE 스캔 상태, MQTT 연결 상태, 보일러 현재 온도, 예약 모드가 한 화면에 같이 있으면 작은 상태 변경 하나에도 카드 여러 개가 다시 그려진다. 처음엔 리빌드가 조금 늘어도 괜찮다고 봤는데, 실제 기기 상태가 1~2초마다 들어오니 화면이 묘하게 무거워졌다.

문제는 Riverpod 자체가 아니라 내가 상태를 너무 크게 보고 있었다는 점이다. `ref.watch(homeDashboardProvider)`로 화면 전체를 구독하면 MQTT ping 시간만 바뀌어도 온도 카드, 전원 버튼, BLE 등록 배너가 전부 build를 다시 탄다. 로그를 찍어보니 사용자는 변화를 못 느끼는데 위젯은 계속 일하고 있었다.

## 화면 전체 상태를 그대로 보지 않는다

대시보드 상태는 보통 이런 식으로 커진다.

```dart
class HomeDashboardState {
  final bool mqttConnected;
  final bool bleScanning;
  final double currentTemp;
  final bool boilerOn;
  final DateTime lastPacketAt;
}
```

이 상태를 루트 화면에서 통째로 watch하면 구현은 편하다.

```dart
final state = ref.watch(homeDashboardProvider);
```

해보니 됐다. 다만 `lastPacketAt`만 바뀌어도 전원 버튼까지 다시 그려졌다. IoT 앱에서는 패킷 수신 시간, RSSI, 연결 품질처럼 자주 바뀌는 값이 많아서 이 방식이 금방 부담이 된다.

## 카드 단위로 select를 건다

전원 버튼은 전원 상태만 보면 된다. MQTT 연결 시간이나 BLE 스캔 여부를 알 필요가 없다.

```dart
class BoilerPowerButton extends ConsumerWidget {
  const BoilerPowerButton({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final boilerOn = ref.watch(
      homeDashboardProvider.select((state) => state.boilerOn),
    );

    return Switch(
      value: boilerOn,
      onChanged: (_) {
        ref.read(homeDashboardProvider.notifier).toggleBoiler();
      },
    );
  }
}
```

온도 카드도 같은 식으로 분리했다.

```dart
final currentTemp = ref.watch(
  homeDashboardProvider.select((state) => state.currentTemp),
);
```

이렇게 바꾸면 MQTT heartbeat가 들어와도 전원 버튼은 다시 그려지지 않는다. 반대로 보일러 전원이 바뀌면 온도 카드도 필요 이상으로 흔들리지 않는다.

## select가 만능은 아니었다

솔직하게 정리해보면 `select`를 아무 데나 붙이면 코드만 지저분해진다. 화면이 작고 상태 변경이 드문 설정 페이지는 통째로 watch해도 충분했다. 내가 기준으로 삼은 건 세 가지였다.

- 같은 화면에 카드가 많다.
- BLE, MQTT, 타이머처럼 상태가 자주 바뀐다.
- 특정 위젯이 상태 일부만 필요로 한다.

이 조건이 아니면 굳이 쪼개지 않았다. 특히 리스트 아이템마다 복잡한 `select`를 넣으면 디버깅할 때 오히려 어느 값이 리빌드를 막는지 헷갈렸다.

## 객체 비교 기준을 조심한다

`select`는 선택한 값이 바뀌었는지 비교한다. 그래서 `List`나 직접 만든 객체를 매번 새로 만들면 기대만큼 리빌드가 줄지 않는다.

```dart
final devices = ref.watch(
  bleScanProvider.select((state) => state.visibleDevices),
);
```

`visibleDevices`가 매번 새 리스트면 내용이 같아도 바뀐 값처럼 보일 수 있다. BLE 스캔 결과는 RSSI만 계속 바뀌는 경우가 많아서, 화면에 필요한 요약 모델을 따로 만들고 중복 기기 병합을 Repository 쪽에서 처리하는 편이 나았다.

짧게 정리하면 이렇다. Riverpod `select`는 Flutter IoT 대시보드에서 리빌드를 줄이는 도구지만, 설계 부족을 덮어주는 기능은 아니다. 상태를 화면 단위로 크게 들고 있다면 카드가 필요한 값만 보게 만든다. 자주 바뀌는 MQTT, Flutter BLE 값은 특히 분리한다. 다만 모든 위젯에 습관처럼 붙이지 말고, 실제로 자주 바뀌는 값과 무거운 위젯에만 적용하는 쪽이 유지보수하기 좋았다.
