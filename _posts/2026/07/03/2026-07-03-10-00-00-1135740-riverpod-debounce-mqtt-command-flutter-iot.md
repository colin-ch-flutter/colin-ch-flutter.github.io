---
layout: post
title: "Flutter IoT Riverpod debounce - MQTT 보일러 명령 연타 막기"
description: "Flutter IoT 앱에서 Riverpod Notifier로 MQTT 보일러 제어 명령을 debounce 처리하고 Flutter 스마트홈 화면의 중복 publish 문제를 줄이는 방법을 정리했다."
date: 2026-07-03
tags: [Flutter, IoT, MQTT, 스마트홈, 보일러]
comments: true
share: true
---
![Flutter IoT Riverpod MQTT debounce 스마트홈 보일러 제어](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

Flutter IoT 앱에서 Flutter 스마트홈 보일러 버튼은 화면상으로는 단순하지만, MQTT publish는 생각보다 쉽게 꼬인다. Riverpod Notifier 안에서 debounce 기준을 잡아두면 사용자가 전원 버튼이나 온도 버튼을 빠르게 눌러도 마지막 의도만 기기에 전달할 수 있다. MQTT 연결 끊김이 섞인 상황에서도 중복 명령 로그를 줄이는 데 도움이 됐다.

처음엔 버튼 `onPressed`에서 바로 `publishCommand()`를 호출했다. 해보니 됐다. 문제는 실제 가족이 앱을 만졌을 때 나왔다. 보일러 전원 버튼을 두세 번 빠르게 누르면 `on`, `off`, `on` 명령이 거의 동시에 나갔다. 기기는 마지막 명령만 처리한 것처럼 보이는데, 앱은 중간 응답까지 받아서 상태가 잠깐 흔들렸다.

GetX 시절에는 Controller에 `Timer? _debounceTimer`를 두고 처리했다. Riverpod으로 옮긴 뒤에는 이 타이머가 화면이 아니라 명령 상태를 가진 Notifier 안에 있어야 더 자연스러웠다.

```dart
final boilerCommandProvider =
    NotifierProvider<BoilerCommandNotifier, BoilerCommandState>(
  BoilerCommandNotifier.new,
);

class BoilerCommandNotifier extends Notifier<BoilerCommandState> {
  Timer? _debounceTimer;

  @override
  BoilerCommandState build() {
    ref.onDispose(() {
      _debounceTimer?.cancel();
    });

    return const BoilerCommandState.idle();
  }

  void requestPower(bool enabled) {
    state = BoilerCommandState.pending(enabled: enabled);

    _debounceTimer?.cancel();
    _debounceTimer = Timer(const Duration(milliseconds: 350), () async {
      await _publishPower(enabled);
    });
  }
}
```

핵심은 버튼 연타를 막기 위해 버튼 자체를 무조건 비활성화하지 않았다는 점이다. 사용자는 가장 나중에 누른 값이 화면에 즉시 반영되길 기대한다. 그래서 UI 상태는 `pending`으로 즉시 바꾸고, 실제 MQTT 명령만 350ms 늦게 보냈다. 체감상 앱은 바로 반응하고, 기기에는 불필요한 publish가 줄어든다.

실제 publish는 Repository를 통해 보냈다.

```dart
Future<void> _publishPower(bool enabled) async {
  final repository = ref.read(boilerRepositoryProvider);

  try {
    await repository.publishPower(enabled);
    state = BoilerCommandState.sent(enabled: enabled);
  } catch (error, stackTrace) {
    state = BoilerCommandState.failed(
      enabled: enabled,
      message: '보일러 명령 전송에 실패했다.',
    );
    ref.read(commandLogProvider.notifier).record(error, stackTrace);
  }
}
```

여기서 조심할 부분은 MQTT 연결 상태다. 연결이 끊긴 상태에서 debounce 타이머가 끝나면 publish가 실패한다. 이때 재시도까지 Notifier에 넣으면 금방 비대해진다. 나는 Notifier는 "마지막 사용자 의도"와 "전송 결과"만 다루고, MQTT 재연결과 큐 처리는 Repository 아래쪽으로 내렸다. UI 제어 흐름과 통신 복구 흐름을 섞지 않기 위해서다.

온도 조절도 같은 방식으로 처리했다. `+`, `-` 버튼을 누를 때마다 바로 MQTT를 보내지 않고 화면 값만 갱신한다. 사용자가 손을 뗀 뒤 짧은 지연 후 마지막 목표 온도만 보낸다. 실제 보일러 패널도 비슷하게 동작해서 UX가 어색하지 않았다.

```dart
void requestTargetTemperature(int temperature) {
  final next = temperature.clamp(5, 35);
  state = BoilerCommandState.pendingTemperature(next);

  _debounceTimer?.cancel();
  _debounceTimer = Timer(const Duration(milliseconds: 500), () {
    _publishTargetTemperature(next);
  });
}
```

테스트에서는 `fakeAsync`로 타이머를 밀어보는 편이 깔끔했다. 버튼을 세 번 누른 뒤 499ms까지는 publish가 없어야 하고, 500ms가 지난 뒤 마지막 값 한 번만 나가야 한다. 이 테스트가 있으면 나중에 UI를 바꿔도 MQTT 명령이 다시 연타로 나가는 회귀를 바로 잡을 수 있다.

주의할 점은 debounce가 모든 명령에 맞지는 않는다는 것이다. 긴급 정지, 도어락 열림 같은 명령은 지연시키면 안 된다. 반대로 보일러 전원, 목표 온도, 조명 밝기처럼 사용자가 값을 조정하는 흐름에는 잘 맞았다.

짧게 남기면 이렇다. Flutter IoT Riverpod 구조에서 debounce는 버튼 위젯보다 명령 Notifier에 두는 편이 오래 버텼다. 화면은 즉시 반응하고, MQTT publish는 마지막 의도만 보낸다. 연결 복구와 재시도는 Repository 쪽 책임으로 남겨야 Flutter 스마트홈 제어 코드가 덜 엉킨다.
