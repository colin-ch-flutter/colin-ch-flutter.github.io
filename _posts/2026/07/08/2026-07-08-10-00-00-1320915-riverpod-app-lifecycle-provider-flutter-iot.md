---
layout: post
title: "Flutter IoT Riverpod AppLifecycle - 백그라운드 복귀 때 MQTT와 BLE 상태 다시 맞추기"
description: "Flutter IoT 앱에서 Riverpod AppLifecycle Provider로 Flutter BLE 스캔과 MQTT 연결 끊김을 분리해 백그라운드 복귀 상태 꼬임을 줄인 기준을 정리했다."
date: 2026-07-08
tags: [Flutter, Riverpod, MQTT, BLE, IoT, 스마트홈]
comments: true
share: true
---
![Flutter IoT Riverpod AppLifecycle](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
이 그림에서는 앱이 백그라운드와 포그라운드를 오갈 때 통신 상태를 한 번 더 검증해야 하는 흐름을 보면 된다.

Flutter IoT 앱에서 백그라운드 복귀는 Riverpod Provider로 따로 빼는 편이 낫다. Flutter 스마트홈 화면은 Flutter BLE 연결 상태와 MQTT 상태를 동시에 보여주는데, 앱을 3분 정도 내려뒀다가 다시 열면 UI는 연결됨인데 실제 소켓은 끊긴 경우가 있었다. `mqtt5_client` 재연결만 믿거나 `flutter_blue_plus` 스캔을 화면 `initState`에 다시 붙이면 한쪽은 살아나고 한쪽은 늦게 따라왔다.

처음엔 GetX 때 쓰던 `WidgetsBindingObserver`를 그대로 `ConsumerStatefulWidget`에 넣었다. 해보니 됐다. 문제는 상세 화면 2개를 오가면 observer가 2번 등록됐고, `resumed` 한 번에 MQTT reconnect와 BLE scan restart가 각각 2번씩 실행됐다. 로그에는 `connected`가 찍히는데 실제 기기 카드 온도는 1초 뒤에 이전 값으로 되돌아갔다.

| 처리 위치 | 괜찮았던 일 | 터졌던 일 |
|---|---|---|
| Widget `initState` | 화면 진입 시 한 번 시작 | 빠른 pop에서 dispose 이후 갱신 |
| Repository 내부 | 통신 코드는 모임 | 앱 상태를 몰라 재시도 타이밍이 둔함 |
| Riverpod lifecycle Provider | 앱 전역 이벤트를 한 곳에서 전달 | 너무 많은 일을 넣으면 전역 Controller가 됨 |

핵심은 앱 생명주기를 값으로 만들고, 통신 Provider가 그 값을 보고 자기 일만 하는 구조다.

```dart
final appLifecycleProvider =
    NotifierProvider<AppLifecycleNotifier, AppLifecycleState>(
  AppLifecycleNotifier.new,
);

class AppLifecycleNotifier extends Notifier<AppLifecycleState>
    with WidgetsBindingObserver {
  @override
  AppLifecycleState build() {
    WidgetsBinding.instance.addObserver(this);
    ref.onDispose(() {
      WidgetsBinding.instance.removeObserver(this);
    });
    return AppLifecycleState.resumed;
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    this.state = state;
  }
}
```

MQTT 쪽은 `resumed`에서 무조건 connect하지 않았다. 마지막 수신 시각이 45초를 넘었거나, 세션 상태가 `disconnected`일 때만 재확인했다.

```dart
final mqttResumeGuardProvider = Provider<void>((ref) {
  ref.listen(appLifecycleProvider, (previous, next) {
    if (next != AppLifecycleState.resumed) return;

    final session = ref.read(mqttSessionProvider);
    if (session.isStale || session.isDisconnected) {
      ref.read(mqttSessionProvider.notifier).reconnect(reason: 'app_resumed');
    }
  });
});
```

BLE는 더 조심해야 했다. iOS에서는 백그라운드 뒤 BLE 연결이 끊기는 일이 있고, Android에서는 스캔을 바로 재시작하면 권한 다이얼로그 상태와 엇갈릴 수 있다. 그래서 복귀 이벤트가 오면 연결된 기기만 health check를 하고, 등록 화면이 열려 있을 때만 스캔을 다시 시작했다. `AppLifecycleState.inactive`에서는 아무것도 하지 않았다. iOS 멀티태스킹 화면을 잠깐 여는 0.2초 구간에서 reconnect를 걸었다가 중복 연결을 만든 적이 있다.

예전 [WidgetsBinding 테스트 글]({% post_url 2026-06-29-10-00-00-938220-flutter-app-lifecycle-test-widgetsbinding-mqtt-reconnect %})에서는 생명주기를 테스트로 검증하는 데 초점을 뒀다. Riverpod 전환 뒤에는 테스트보다 경계가 더 문제였다. observer는 하나만 등록하고, MQTT와 BLE는 각자 Provider에서 반응하게 만들어야 화면 수명과 통신 수명이 섞이지 않는다.

짧게 남기면 이렇다.

- 앱 생명주기는 Widget마다 붙이지 말고 Riverpod Provider 하나로 올린다.
- `resumed`는 reconnect 명령이 아니라 상태 재검사 신호로 본다.
- MQTT는 마지막 수신 시각과 세션 상태를 보고 재연결한다.
- BLE 스캔 재시작은 화면 상태와 권한 상태를 같이 확인한다.
- `inactive`에서 통신을 끊으면 iOS에서 짧은 전환에도 연결이 흔들릴 수 있다.
