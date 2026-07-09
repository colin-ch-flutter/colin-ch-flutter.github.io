---
layout: post
title: "Flutter IoT Riverpod commandId - MQTT 중복 명령을 멱등하게 막기"
description: "Flutter IoT 앱에서 Riverpod과 MQTT commandId로 Flutter 스마트홈 보일러 제어 명령이 중복 실행되는 문제를 줄인 기준을 정리했다."
date: 2026-07-10
tags: [Flutter, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod MQTT commandId 멱등 처리](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)
이 그림에서는 앱 버튼, MQTT 브로커, 기기 펌웨어 사이에서 같은 명령이 두 번 흐를 수 있다는 점을 봐야 한다.

Flutter IoT 앱에서 Flutter 스마트홈 보일러 제어는 `commandId`를 붙여 멱등하게 처리하는 편이 안전했다. Riverpod Optimistic UI와 MQTT ACK 타임아웃을 넣어도, 네트워크 재시도나 화면 재진입 때문에 같은 publish가 두 번 나갈 수 있다. 처음엔 debounce만 있으면 충분하다고 봤는데 아니었다. 사용자가 한 번 누른 명령과 앱이 복구 중 다시 보낸 명령을 구분할 값이 필요했다.

문제는 실패처럼 보이는 성공이었다. LTE에서 publish 직후 앱이 timeout을 냈고, 사용자는 재시도 버튼을 눌렀다. 그런데 기기는 첫 번째 명령을 이미 처리한 상태였다. 두 번째 publish까지 들어가면서 보일러 전원이 `ON -> OFF`처럼 보이는 로그가 남았다. 실제로는 토글 명령을 보낸 설계가 더 큰 문제였다.

| 기준 | 나쁜 처리 | 나은 처리 |
|---|---|---|
| 전원 | toggle | `setPower(on: true, commandId)` |
| 목표 온도 | `+1`, `-1` | `setTargetTemp(23, commandId)` |
| ACK 매칭 | 마지막 reported 값 | 같은 `commandId` 또는 보낸 시각 이후 값 |
| 재시도 | 새 명령 생성 | 같은 의도면 같은 id 유지 |

핵심은 UI 명령을 만들 때부터 id를 같이 들고 가는 것이다.

```dart
class DeviceCommand {
  DeviceCommand({
    required this.commandId,
    required this.deviceId,
    required this.powerOn,
    required this.createdAt,
  });

  final String commandId;
  final String deviceId;
  final bool powerOn;
  final DateTime createdAt;
}

final boilerCommandProvider =
    NotifierProvider<BoilerCommandNotifier, AsyncValue<void>>(
  BoilerCommandNotifier.new,
);

class BoilerCommandNotifier extends Notifier<AsyncValue<void>> {
  @override
  AsyncValue<void> build() => const AsyncData(null);

  Future<void> setPower(String deviceId, bool powerOn) async {
    final command = DeviceCommand(
      commandId: '${deviceId}_${DateTime.now().microsecondsSinceEpoch}',
      deviceId: deviceId,
      powerOn: powerOn,
      createdAt: DateTime.now(),
    );

    state = const AsyncLoading();
    await ref.read(mqttCommandRepositoryProvider).publish(command);
    state = const AsyncData(null);
  }
}
```

여기서 헷갈렸던 건 `commandId`를 매번 새로 만들면 멱등 처리가 된다고 착각한 부분이다. 새 버튼 클릭은 새 id가 맞다. 하지만 ACK timeout 뒤 같은 사용자 의도를 다시 보내는 재시도라면 기존 id를 유지해야 한다. 그래야 서버나 펌웨어가 “이미 처리한 명령”으로 판단할 수 있다.

펌웨어나 서버가 `commandId`를 지원하지 않으면 앱만으로 완벽하게 막기는 어렵다. 그래도 토글 대신 원하는 최종 상태를 보내는 것만으로 위험이 꽤 줄어든다. `togglePower`는 두 번 실행되면 원래 상태로 돌아가지만, `setPower(true)`는 두 번 실행돼도 결과가 같다.

[MQTT ACK 타임아웃]({% post_url 2026-07-09-14-00-00-1394985-riverpod-mqtt-command-ack-timeout-flutter-iot %})을 붙일 때도 이 기준이 필요했다. timeout은 “실패 확정”이 아니라 “확인 실패”일 수 있다. 그래서 Flutter BLE 근접 제어나 MQTT 제어처럼 실제 기기를 움직이는 기능은 commandId, 보낸 시각, 최종 상태 명령을 같이 남겨야 로그가 읽힌다.

짧게 정리하면 이렇다. Flutter IoT Riverpod 구조에서 MQTT 중복 명령은 debounce만으로 끝나지 않는다. Flutter 스마트홈 제어 명령에는 `commandId`를 붙이고, 토글보다 최종 상태를 보내고, 같은 재시도에서는 같은 id를 유지한다. 이 세 가지를 지키면 ACK 지연과 앱 재시작이 섞여도 보일러가 엉뚱하게 두 번 움직이는 일을 줄일 수 있다.
