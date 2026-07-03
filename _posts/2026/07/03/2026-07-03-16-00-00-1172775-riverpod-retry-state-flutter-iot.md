---
layout: post
title: "Flutter IoT Riverpod retry - 재시도 버튼 상태를 통신 세션과 분리하기"
description: "Flutter IoT 앱에서 Riverpod으로 MQTT·BLE 재시도 버튼 상태를 통신 세션과 분리해 중복 요청을 막는 방법을 정리했다."
date: 2026-07-03
tags: [Flutter, Riverpod, MQTT, BLE, IoT]
comments: true
share: true
---
![Flutter IoT Riverpod retry 상태 분리](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

이 그림에서는 사용자의 재시도 버튼과 실제 네트워크 세션이 같은 속도로 움직이지 않는다는 점을 봐야 한다.

Flutter IoT 앱에서 재시도 버튼은 통신 세션 자체가 아니라 “사용자가 다시 시도했다”는 짧은 UI 상태로 다루는 편이 낫다. MQTT 연결, BLE 재연결, 대시보드 새로고침을 버튼 하나에 묶으면 실패했을 때 어디가 막혔는지 흐려진다. 나는 이걸 처음엔 단순하게 봤다가, 같은 보일러에 publish가 두 번 나가는 버그를 만들었다.

문제는 버튼 비활성화 기준이었다. `isConnecting` 하나로 모든 재시도 버튼을 막았더니 BLE 재연결 중에는 MQTT 새로고침도 눌리지 않았다. 반대로 버튼마다 상태를 따로 두지 않으면 사용자가 0.5초 간격으로 탭할 때 같은 요청이 중복 실행됐다.

| 상태 | 위치 | 수명 |
|---|---|---:|
| MQTT 연결 상태 | Repository/Session Provider | 앱 세션 |
| BLE 재연결 상태 | deviceId별 Provider | 기기 단위 |
| 버튼 처리 중 | 화면 Notifier | 요청 1회 |
| 에러 메시지 표시 | Widget `ref.listen` | 이벤트 1회 |

버튼은 짧게 잠그고, 실제 재연결은 아래 계층에 맡긴다.

```dart
final retryButtonProvider =
    AutoDisposeNotifierProvider<RetryButtonNotifier, bool>(
  RetryButtonNotifier.new,
);

class RetryButtonNotifier extends AutoDisposeNotifier<bool> {
  @override
  bool build() => false;

  Future<void> run(Future<void> Function() action) async {
    if (state) return;
    state = true;

    try {
      await action();
    } finally {
      state = false;
    }
  }
}
```

화면에서는 버튼 잠금만 읽고, 어떤 작업을 재시도할지는 명확히 넘긴다.

```dart
ElevatedButton(
  onPressed: ref.watch(retryButtonProvider)
      ? null
      : () {
          ref.read(retryButtonProvider.notifier).run(() {
            return ref.read(mqttSessionProvider.notifier).reconnect();
          });
        },
  child: const Text('다시 연결'),
)
```

여기서 헷갈렸던 건 “재시도 버튼이니까 reconnect를 직접 갖고 있어도 되지 않나”라는 생각이었다. 작게 만들 때는 된다. 그런데 화면이 3개로 늘고, BLE 기기 카드마다 재시도 버튼이 붙으면 화면 Notifier가 통신 정책을 다 알아야 한다. 그때부터 테스트가 어려워진다.

주의할 점은 버튼 잠금 시간이 너무 길어지면 안 된다는 것이다. 네트워크 재시도는 5초, 10초로 늘어날 수 있지만 버튼 UI는 사용자가 상태를 이해할 수 있게 빠르게 풀리거나 별도 진행 상태를 보여줘야 한다. 내가 확인한 기준으론 1회 요청 잠금과 백오프 재시도 상태를 나눴을 때 로그가 가장 읽기 쉬웠다.

짧게 정리하면 이렇다.

- Flutter IoT 재시도 버튼은 통신 세션이 아니라 UI 요청 상태다.
- Riverpod Notifier는 중복 탭 방지만 맡기고, 재연결 정책은 Repository로 내린다.
- MQTT와 BLE 재시도 상태를 하나의 `isConnecting`으로 묶으면 화면이 답답해진다.
- 버튼 잠금, 에러 표시, 실제 재시도는 서로 다른 수명으로 관리한다.
