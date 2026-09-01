---
layout: post
title: "Flutter ValueListenableBuilder 실전 - MQTT 상태 갱신과 불필요한 rebuild 줄이기"
description: "Flutter IoT 대시보드에서 MQTT 상태가 바뀔 때 전체 화면이 rebuild되는 문제를 ValueListenableBuilder와 ValueNotifier로 분리하는 방법을 정리했다."
date: 2026-09-02
tags: [Flutter, Dart, MQTT, IoT, 성능최적화]
comments: true
share: true
---

![Flutter ValueListenableBuilder로 MQTT 상태 갱신 범위 줄이기](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=1200&q=80)

MQTT 메시지가 들어올 때마다 Flutter IoT 대시보드 전체가 rebuild된다면 `ValueListenableBuilder`부터 검토할 만하다. 상태 하나를 `ValueNotifier`로 쪼개고 해당 카드만 구독하게 만들면, 보일러 온도 하나가 바뀌어도 화면 전체의 `build()`를 다시 실행하지 않아도 된다.

처음엔 GetX Controller에서 `update()`를 호출하면 충분한 줄 알았다. 그런데 기기 12개의 상태를 한 Controller에서 관리하니 온도 센서 하나의 메시지에도 차트, 방 목록, 다른 기기 카드까지 같이 그려졌다. 프레임 드랍 로그를 따라가 보니 MQTT 처리보다 위젯 rebuild 비용이 더 컸다.

## 상태를 화면 단위로 쪼갠다

`ValueNotifier`는 값이 바뀔 때 리스너에게 알림을 보내는 작은 상태 저장소다. 전체 모델을 전달하지 않고 카드가 필요한 값만 노출하는 게 핵심이다.

```dart
class DeviceTileState {
  final String deviceId;
  final double temperature;
  final bool powerOn;

  const DeviceTileState({
    required this.deviceId,
    required this.temperature,
    required this.powerOn,
  });
}

class DeviceTileController {
  final state = ValueNotifier<DeviceTileState>(
    const DeviceTileState(
      deviceId: 'boiler-01',
      temperature: 20,
      powerOn: false,
    ),
  );

  void onMqttMessage(DeviceTileState next) {
    if (state.value.temperature == next.temperature &&
        state.value.powerOn == next.powerOn) {
      return;
    }
    state.value = next;
  }

  void dispose() => state.dispose();
}
```

같은 값이 반복해서 들어오는 MQTT 메시지는 여기서 걸러낸다. 브로커가 retained message를 재전송하거나 장치가 1초마다 같은 온도를 보내는 경우에 효과가 있었다.

## 카드만 다시 그리게 만든다

대시보드의 상위 위젯은 `ValueListenableBuilder`를 만들고, 구독 해제는 Controller의 생명주기에 맞춘다.

```dart
class DeviceTile extends StatelessWidget {
  final DeviceTileController controller;

  const DeviceTile({super.key, required this.controller});

  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<DeviceTileState>(
      valueListenable: controller.state,
      builder: (context, value, child) {
        return ListTile(
          title: Text(value.deviceId),
          subtitle: Text('${value.temperature.toStringAsFixed(1)}°C'),
          trailing: Switch(
            value: value.powerOn,
            onChanged: (_) {},
          ),
        );
      },
    );
  }
}
```

| 패턴 | 상태 변경 시 다시 그리는 범위 | 잘 맞는 경우 |
|---|---|---|
| 상위 `setState` | 화면 하위 트리 전체 | 작은 화면 |
| GetX `update()` | Controller에 연결된 위젯 | 기존 GetX 구조 |
| `ValueListenableBuilder` | 특정 값 구독 영역 | 카드·센서 값 분리 |

이 그림에서 봐야 할 부분은 상태 관리 도구의 우열이 아니라 rebuild 경계다. 이미 GetX를 쓰는 프로젝트라면 전부 바꿀 필요 없이, 메시지가 자주 들어오는 센서 카드부터 `ValueNotifier`로 분리하면 된다.

## dispose와 equality를 빼먹으면 생기는 문제

`ValueNotifier`를 만들었으면 반드시 `dispose()`해야 한다. 화면을 열고 닫을 때마다 새 Controller를 만들면서 정리하지 않으면 리스너가 남거나 오래된 카드가 계속 갱신된다. GetX라면 `onClose()`에서 호출한다.

또 `DeviceTileState`에 `==` 비교를 구현하지 않으면 객체가 새로 만들어질 때마다 같은 값도 변경으로 인식한다. 단순한 숫자 비교로 시작해도 되지만 필드가 늘어나면 `Equatable` 같은 비교 전략을 적용하는 편이 낫다.

짧게 정리하면, MQTT 메시지 처리와 위젯 rebuild를 한 덩어리로 묶지 않는 게 기준이다. 값이 자주 바뀌는 IoT 카드에는 `ValueNotifier`로 상태를 좁히고, `ValueListenableBuilder`로 화면 경계를 만들고, 동일 값 필터와 `dispose()`까지 같이 넣어야 실제 성능 개선으로 이어진다.
