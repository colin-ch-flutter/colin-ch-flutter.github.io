---
layout: post
title: "Flutter IoT Riverpod Command Notifier - 스마트홈 보일러 전원 토글 상태 분리하기"
description: "Flutter IoT 앱에서 Riverpod Command Notifier로 Flutter 스마트홈 보일러 전원 토글과 MQTT publish 상태를 읽기 상태와 분리한 기준을 정리했다."
date: 2026-07-08
tags: [Flutter, Riverpod, MQTT, BLE, IoT, 스마트홈, CleanArchitecture]
comments: true
share: true
---

![Flutter IoT Riverpod Command Notifier 보일러 토글](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)
이 그림에서는 사용자가 누른 전원 토글과 실제 기기 상태 동기화가 같은 문제가 아니라는 점을 봐야 한다.

Flutter IoT 앱에서 Flutter 스마트홈 보일러 전원 토글은 `AsyncValue` 하나로 처리하면 금방 꼬인다. 화면은 MQTT로 들어온 실제 전원 상태를 보여줘야 하고, 버튼은 사용자가 방금 누른 명령의 진행 상태를 보여줘야 한다. Flutter BLE 근접 연결이나 MQTT 세션 상태까지 같이 얹히면 더 헷갈린다. 내가 확인한 기준으론 읽기 상태는 Stream/Future Provider, 쓰기 명령은 별도 Command Notifier로 나누는 쪽이 오래 버텼다.

처음엔 `DeviceCardState` 안에 `isPowerChanging`을 같이 넣었다. 온도, 전원, 연결 상태, 버튼 로딩을 한 모델에 넣으니 화면 코드는 짧았다. 문제는 실패했을 때였다. MQTT publish는 실패했는데 기기에서 직전 retained 메시지가 다시 들어오면 카드가 성공한 것처럼 보였다. 반대로 실제 전원은 켜졌는데 ACK가 늦으면 버튼만 계속 돌았다.

내가 나눈 기준은 이렇다.

| 상태 | 소유자 | 갱신 기준 |
| --- | --- | --- |
| 현재 전원값 | MQTT 구독 Provider | 기기 reported 메시지 |
| 버튼 pending | Command Notifier | 사용자 탭부터 publish 종료까지 |
| 실패 메시지 | Command Notifier | publish 예외나 timeout |
| BLE 근접 여부 | BLE Provider | 스캔/연결 이벤트 |

쓰기 명령은 화면 모델과 분리해서 둔다. 이 코드는 낙관적 UI를 과하게 밀지 않고, 버튼 중복 탭만 막는 형태다.

```dart
final powerCommandProvider =
    NotifierProvider.family<PowerCommandNotifier, PowerCommandState, String>(
  PowerCommandNotifier.new,
);

class PowerCommandState {
  const PowerCommandState({this.pending = false, this.error});

  final bool pending;
  final Object? error;
}

class PowerCommandNotifier
    extends FamilyNotifier<PowerCommandState, String> {
  @override
  PowerCommandState build(String deviceId) {
    return const PowerCommandState();
  }

  Future<void> toggle(bool nextPower) async {
    if (state.pending) return;

    state = const PowerCommandState(pending: true);

    try {
      await ref.read(mqttCommandRepositoryProvider).publishPower(
            deviceId: arg,
            power: nextPower,
          );
      state = const PowerCommandState();
    } catch (e) {
      state = PowerCommandState(error: e);
    }
  }
}
```

카드는 읽기 상태와 명령 상태를 따로 본다. 여기서 중요한 건 `power` 값을 버튼 pending으로 직접 바꾸지 않는 것이다. 실제 전원값은 기기 reported 메시지가 바꾼다.

```dart
class BoilerPowerSwitch extends ConsumerWidget {
  const BoilerPowerSwitch({super.key, required this.deviceId});

  final String deviceId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final device = ref.watch(deviceStatusProvider(deviceId));
    final command = ref.watch(powerCommandProvider(deviceId));

    return device.when(
      data: (status) => Switch(
        value: status.power,
        onChanged: command.pending
            ? null
            : (value) {
                ref
                    .read(powerCommandProvider(deviceId).notifier)
                    .toggle(value);
              },
      ),
      loading: () => const SizedBox(width: 48, height: 48),
      error: (_, __) => const Icon(Icons.error_outline),
    );
  }
}
```

여기서 헷갈렸던 건 “사용자가 눌렀으니 바로 켜진 것처럼 보여줘야 하지 않나”였다. 보일러 앱에서는 그게 항상 맞지 않았다. 로컬 네트워크에서는 300ms 안에 reported 메시지가 오지만, LTE 환경에서 AWS IoT Core를 거치면 1~2초까지 벌어졌다. 그 사이에 낙관적 UI를 켰다가 실패하면 사용자는 실제 기기가 켜졌다고 착각한다.

그래서 나는 전원값 자체는 보수적으로 두고, 버튼 주변에만 진행 상태를 표시했다. UX가 조금 답답해 보여도 운영 로그는 훨씬 읽기 쉬웠다. “사용자가 명령을 보냄”, “MQTT publish 성공”, “기기 reported 반영”이 서로 다른 시점으로 남기 때문이다.

주의할 점은 Command Notifier가 MQTT 재연결까지 책임지면 안 된다는 것이다. 연결이 끊겼으면 Repository가 명확한 예외를 던지고, 화면은 실패를 보여주면 된다. 재연결 정책까지 버튼 상태에 넣으면 같은 publish가 두 번 나가거나, 앱 복귀 시점에 오래된 명령이 다시 실행될 수 있다.

짧게 정리하면 이렇다. Flutter IoT에서 Riverpod으로 Flutter 스마트홈 제어 화면을 만들 때 읽기 상태와 쓰기 명령 상태는 수명이 다르다. MQTT reported 값은 실제 기기 상태이고, Command Notifier는 사용자의 요청 상태다. 둘을 분리하면 보일러 전원 토글, Flutter BLE 근접 제어, 오프라인 실패 표시를 같은 카드 안에서도 덜 헷갈리게 다룰 수 있다.
