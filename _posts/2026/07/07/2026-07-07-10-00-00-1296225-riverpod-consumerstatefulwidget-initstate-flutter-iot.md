---
layout: post
title: "Flutter IoT Riverpod ConsumerStatefulWidget - initState에서 BLE 초기화 꼬임 막기"
description: "Flutter IoT 앱에서 Riverpod ConsumerStatefulWidget으로 Flutter BLE 초기화와 MQTT 상태 구독을 분리해 initState 부작용이 꼬이는 문제를 정리했다."
date: 2026-07-07
tags: [Flutter, Riverpod, BLE, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod ConsumerStatefulWidget 초기화](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
이 그림에서는 화면 생명주기와 BLE/MQTT 통신 생명주기를 같은 것으로 보면 안 된다는 점만 보면 된다.

Flutter IoT 앱에서 Flutter 스마트홈 기기 상세 화면을 Riverpod으로 옮길 때 `ConsumerWidget`만 고집하면 오히려 꼬인다. Flutter BLE 연결 준비, MQTT 상태 구독, 화면 진입 로그처럼 한 번만 실행해야 하는 작업은 `ConsumerStatefulWidget`으로 분리하는 편이 낫다. 처음엔 `build()` 안에서 `ref.watch()`와 `ref.read()`를 섞어도 된다고 봤는데, 보일러 상세 화면을 왕복하니 BLE 초기화가 2번씩 찍혔다.

문제는 GetX 습관이었다. 예전에는 Controller의 `onInit()`에 스캔 준비, MQTT subscribe, analytics 로그를 같이 넣었다. Riverpod으로 바꾸면서 화면을 전부 `ConsumerWidget`으로 만들었더니 “화면이 그려지는 일”과 “통신을 시작하는 일”이 다시 한 파일에 섞였다. 특히 `build()`는 생각보다 자주 돈다. 부모 탭이 바뀌거나 온도 값 하나가 갱신돼도 다시 호출된다.

| 작업 | 적당한 위치 | 이유 |
|---|---|---|
| 화면 상태 표시 | `ConsumerWidget` + `ref.watch` | 값이 바뀌면 다시 그려야 한다 |
| 버튼 명령 | 콜백 안 `ref.read` | 사용자가 누른 순간만 실행한다 |
| 진입 시 1회 초기화 | `ConsumerState.initState` | build 반복과 분리한다 |
| 상태 변화에 따른 SnackBar | `ref.listen` | 화면 그리기와 부작용을 분리한다 |

BLE 준비처럼 화면 진입 시 한 번만 필요한 코드는 `ConsumerStatefulWidget`에서 다루는 쪽이 명확했다.

```dart
class DeviceDetailPage extends ConsumerStatefulWidget {
  const DeviceDetailPage({super.key, required this.deviceId});

  final String deviceId;

  @override
  ConsumerState<DeviceDetailPage> createState() => _DeviceDetailPageState();
}

class _DeviceDetailPageState extends ConsumerState<DeviceDetailPage> {
  @override
  void initState() {
    super.initState();
    Future.microtask(() {
      ref.read(bleSessionProvider.notifier).prepare(widget.deviceId);
    });
  }

  @override
  Widget build(BuildContext context) {
    final status = ref.watch(deviceCardProvider(widget.deviceId));
    return DeviceStatusView(status: status);
  }
}
```

여기서 일부러 `ref.watch`를 쓰지 않는다. 초기화는 구독이 아니라 명령이다. `watch`로 읽으면 상태 변경 때마다 의도가 흐려지고, 나중에 누가 봐도 “이 Provider를 왜 구독하지?”라는 질문이 남는다.

내가 한 번 더 헷갈렸던 건 `initState` 안에서 바로 async 작업을 길게 붙인 부분이다. BLE 권한 요청, 주변 기기 캐시 조회, MQTT topic 확인을 한 메서드에 넣었더니 화면을 빠르게 닫을 때 dispose 이후 상태 갱신 로그가 나왔다. 그래서 `prepare()`는 짧게 시작 신호만 보내고, 실제 Stream 구독과 정리는 Provider 내부 `ref.onDispose`로 옮겼다.

```dart
final bleSessionProvider =
    NotifierProvider<BleSessionNotifier, BleSessionState>(
  BleSessionNotifier.new,
);

class BleSessionNotifier extends Notifier<BleSessionState> {
  @override
  BleSessionState build() {
    ref.onDispose(() {
      ref.read(bleRepositoryProvider).stopPrepare();
    });
    return const BleSessionState.idle();
  }

  Future<void> prepare(String deviceId) async {
    state = const BleSessionState.preparing();
    await ref.read(bleRepositoryProvider).prepare(deviceId);
    state = const BleSessionState.ready();
  }
}
```

[GetX 상태관리]({% post_url 2026-05-02-09-21-39-037035-getx-state-management-routing-flutter-iot %})에서는 Controller 생명주기에 많이 기대도 앱이 작을 땐 버텼다. 하지만 [클린 아키텍처 구조]({% post_url 2026-05-01-11-14-26-024690-clean-architecture-large-flutter-project-structure %})로 Repository 경계를 세운 뒤에는 화면이 통신 수명을 직접 잡는 게 더 위험했다.

체크리스트로 보면 기준은 단순하다.

- `build()` 안에서는 화면에 필요한 값만 `ref.watch`한다.
- 버튼, pull-to-refresh, 재시도처럼 사용자 행동은 콜백에서 `ref.read`한다.
- 화면 진입 시 1회 작업은 `ConsumerStatefulWidget`의 `initState`로 보낸다.
- BLE/MQTT 구독 해제는 화면 dispose보다 Provider `ref.onDispose`에 둔다.
- `initState` 안에서 오래 걸리는 async 흐름을 직접 끌고 가지 않는다.

짧게 정리하면 이렇다. Flutter IoT 앱에서 Riverpod 전환은 모든 화면을 `ConsumerWidget`으로 바꾸는 작업이 아니다. Flutter BLE 초기화나 MQTT 준비처럼 한 번만 실행해야 하는 작업은 `ConsumerStatefulWidget`으로 build와 분리한다. 다만 통신 세션의 실제 수명은 Provider와 Repository가 갖게 해야 한다. 화면은 진입 신호만 주고, 정리는 Provider가 맡는 구조가 덜 흔들렸다.
