---
layout: post
title: "Flutter IoT Riverpod Optimistic UI - 스마트홈 제어 실패 때 롤백 기준 잡기"
description: "Flutter IoT 앱에서 Riverpod으로 Flutter 스마트홈 보일러 제어 Optimistic UI와 MQTT 실패 롤백 기준을 나눈 과정을 정리했다."
date: 2026-07-10
tags: [Flutter, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod Optimistic UI 롤백](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)
이 그림에서는 앱 화면의 빠른 반응과 실제 기기 상태 확인 사이에 시간 차이가 생기는 지점을 봐야 한다.

Flutter IoT 앱에서 Riverpod으로 Flutter 스마트홈 제어를 만들 때 Optimistic UI는 값마다 다르게 적용해야 했다. 보일러 전원은 바로 켜진 것처럼 바꾸지 않고, 목표 온도는 임시값을 보여준 뒤 MQTT reported 값으로 확정하는 쪽이 덜 위험했다. 처음엔 모든 토글과 스텝 버튼을 같은 방식으로 처리했는데, 실패했을 때 사용자가 실제 기기가 동작했다고 착각하는 문제가 있었다.

## 전원과 온도는 실패 비용이 다르다

처음엔 “사용자가 누르면 화면도 바로 바뀌어야 빠르다”고 생각했다. 해보니 반만 맞았다. 목표 온도 `21도 -> 22도`는 실패해도 다시 되돌리면 된다. 하지만 전원 OFF 상태의 보일러를 ON처럼 표시했다가 MQTT 연결 끊김으로 실패하면 이야기가 달라진다. 사용자는 난방이 켜졌다고 믿고 앱을 닫을 수 있다.

내가 확인한 기준은 이렇게 나눴다.

| 제어값 | 화면 반응 | 확정 기준 | 실패 처리 |
|---|---|---|---|
| 전원 ON/OFF | `켜는 중`, `끄는 중` 표시 | MQTT reported 값 | 이전 전원값 유지 |
| 목표 온도 | 임시 온도 즉시 표시 | reported 온도 또는 commandId ack | 이전 온도로 롤백 |
| 예약 모드 | pending 배지 | 서버 저장 성공 + 기기 반영 | 저장 실패 안내 |

전원은 상태 착각이 위험하고, 온도는 조작 빈도가 높다. 그래서 둘을 같은 `pending` 플래그로 묶으면 UX가 애매해졌다.

## Riverpod Notifier에는 임시값과 기준값을 같이 둔다

이 코드는 목표 온도만 제한적으로 낙관적 UI를 적용하고, 실패하면 이전 reported 값으로 되돌리는 구조다.

```dart
class TemperatureCommandState {
  const TemperatureCommandState({
    required this.reported,
    this.optimistic,
    this.error,
  });

  final int reported;
  final int? optimistic;
  final Object? error;

  int get visible => optimistic ?? reported;
}

class TemperatureCommandNotifier extends Notifier<TemperatureCommandState> {
  @override
  TemperatureCommandState build() {
    final device = ref.watch(deviceStateProvider);
    return TemperatureCommandState(reported: device.targetTemperature);
  }

  Future<void> change(int next) async {
    final previous = state.reported;
    state = TemperatureCommandState(
      reported: previous,
      optimistic: next,
    );

    try {
      await ref.read(mqttCommandRepositoryProvider).setTargetTemperature(next);
    } catch (error) {
      state = TemperatureCommandState(
        reported: previous,
        error: error,
      );
    }
  }

  void applyReported(int temperature) {
    state = TemperatureCommandState(reported: temperature);
  }
}
```

핵심은 `visible` 값과 `reported` 값을 섞지 않는 것이다. Widget은 `visible`을 보여주지만, 실제 기기 상태 판단은 계속 `reported`를 본다. 여기서 한 번 실수했다. 임시 온도를 캐시에 저장했더니 앱 재실행 뒤에도 실패한 값이 남았다. Optimistic 값은 메모리에만 두고, MQTT reported가 들어온 뒤에만 Realm 캐시에 저장하는 편이 맞았다.

## 롤백은 조용히 하면 안 된다

롤백 자체보다 사용자가 왜 되돌아갔는지 아는 게 더 중요했다. 온도를 `22도`로 올렸는데 2초 뒤 `21도`로 돌아가면 앱이 이상해 보인다. 그래서 작은 에러 문구와 마지막 시도 시간을 같이 보여줬다.

```dart
final visibleTemp = ref.watch(
  temperatureCommandProvider.select((state) => state.visible),
);

final error = ref.watch(
  temperatureCommandProvider.select((state) => state.error),
);
```

`select`를 쓴 이유는 단순하다. 온도 숫자만 바뀔 때 카드 전체가 다시 그려질 필요는 없었다. MQTT 상태, BLE 근접 상태, 에러 배지가 한 카드에 같이 있으면 작은 변경에도 리빌드가 많아진다.

주의할 점도 있다. 낙관적 UI는 네트워크 실패를 숨기는 장치가 아니다. LTE 환경에서 MQTT reported가 1~2초 늦게 오고, AWS IoT Core 정책 오류가 있으면 publish 자체가 실패한다. 이 경우엔 “조금 기다리면 되겠지”가 아니라 실패 상태를 보여줘야 한다. 특히 SigV4 서명이 틀린 403은 앱에서 재시도해도 해결되지 않는다.

짧게 정리하면 이렇다. Flutter IoT Riverpod 구조에서 Optimistic UI는 모든 제어에 한 번에 넣으면 위험하다. Flutter 스마트홈 보일러 전원은 pending 표시로 두고, 목표 온도처럼 실패 비용이 낮은 값만 임시 반영한다. 임시값은 캐시에 저장하지 않고, MQTT reported 값으로만 확정한다. 이 기준을 세우니 화면은 빨라졌고, 실패했을 때 거짓 상태를 보여주는 일도 줄었다.
