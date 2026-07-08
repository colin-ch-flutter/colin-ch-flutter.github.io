---
layout: post
title: "Flutter BLE Riverpod StreamNotifier - 스마트홈 기기 연결 이벤트 정리하기"
description: "Flutter IoT 앱에서 Flutter BLE 재연결 상태를 Riverpod StreamNotifier로 정리해 화면 이동과 연결 이벤트 꼬임을 줄인 방법이다."
date: 2026-07-08
tags: [Flutter, Riverpod, BLE, flutter_blue_plus, IoT, 스마트홈]
comments: true
share: true
---

![Flutter BLE Riverpod StreamNotifier 연결 상태](https://images.unsplash.com/photo-1558346490-a72e53ae2d4f?w=800&q=80)
이 그림에서는 BLE 기기 연결 상태가 화면이 아니라 별도 상태 흐름으로 관리되어야 한다는 점을 보면 된다.

Flutter IoT 앱에서 Flutter BLE 재연결은 Riverpod `StreamNotifier`로 빼는 편이 관리하기 쉬웠다. Flutter 스마트홈 기기 상세 화면에서 `flutter_blue_plus 사용법`대로 `device.connectionState.listen()`을 바로 붙이면 처음엔 된다. 문제는 화면을 나갔다 들어오거나, 앱이 백그라운드에서 돌아오거나, 사용자가 연결 버튼을 두 번 누르는 순간부터였다. 연결 이벤트가 Widget, Controller, Repository에 흩어지면 어디서 끊겼는지 찾기 어렵다.

[BLE 재연결 전략]({% post_url 2026-05-11-09-24-36-148140-ble-connection-stability-reconnect-error-handling %})을 처음 정리할 때는 GetX Controller 안에서 subscription을 잡았다. 그때는 화면 수명이 곧 연결 수명이라고 생각했다. 해보니 아니었다. BLE 연결은 화면보다 오래 살아야 할 때도 있고, 반대로 기기 등록 화면처럼 화면을 닫으면 바로 정리해야 할 때도 있다.

## StreamProvider만으로는 애매했던 지점

단순 표시라면 `StreamProvider.family`로 충분하다. 하지만 연결 버튼, 수동 재시도, 자동 재연결 횟수 같은 명령이 붙으면 `StreamProvider`만으로는 상태를 모으기 애매했다. 그래서 연결 이벤트를 읽으면서도 명령 메서드를 둘 수 있는 `StreamNotifier`를 썼다.

| 방식 | 맞는 상황 | 내가 겪은 한계 |
|---|---|---|
| `StreamProvider` | 스캔 목록, 단순 연결 표시 | reconnect 명령 위치가 애매함 |
| `Notifier` | 버튼 로딩, 단발 명령 | BLE 연결 Stream 반영이 번거로움 |
| `StreamNotifier` | 연결 이벤트 + 재시도 명령 | 수명 기준을 명확히 정해야 함 |

BLE 연결 이벤트와 수동 재연결을 한 Provider에 묶은 형태다.

```dart
final bleConnectionProvider = StreamNotifierProvider.autoDispose
    .family<BleConnectionNotifier, BleConnectionState, String>(
  BleConnectionNotifier.new,
);

class BleConnectionNotifier
    extends AutoDisposeFamilyStreamNotifier<BleConnectionState, String> {
  late final BleRepository _repository;

  @override
  Stream<BleConnectionState> build(String deviceId) {
    _repository = ref.watch(bleRepositoryProvider);

    ref.onDispose(() {
      _repository.stopReconnect(deviceId);
    });

    return _repository.watchConnection(deviceId);
  }

  Future<void> reconnect() async {
    final deviceId = arg;
    state = const AsyncLoading();

    try {
      await _repository.reconnect(deviceId);
    } catch (e, st) {
      state = AsyncError(e, st);
    }
  }
}
```

여기서 핵심은 `watchConnection()`과 `reconnect()`를 같은 Repository로 모으되, 실제 BLE API 호출은 Provider 안에 직접 넣지 않는 것이다. `flutter_blue_plus`의 `BluetoothDevice`를 Widget까지 끌고 오면 테스트도 어렵고, iOS 실기기에서만 재현되는 끊김을 Fake로 고정하기도 힘들다.

## autoDispose 기준을 화면별로 나눈다

내가 확인한 기준으론 기기 등록 화면은 `autoDispose`가 맞다. 스캔과 임시 연결은 화면을 닫으면 정리해야 한다. 반대로 홈 대시보드의 대표 기기 연결 상태는 너무 빨리 버리면 안 된다. 탭 이동 1번에 BLE reconnect가 다시 돌면 배터리도 낭비되고, 사용자는 보일러 카드가 1초씩 흔들리는 걸 본다.

```text
등록 화면: autoDispose O, 화면 종료 시 stopScan/stopReconnect
상세 화면: autoDispose O, deviceId 단위로 짧게 유지
홈 요약: 세션 Provider에서 유지, 화면 모델만 구독
```

처음엔 모든 BLE Provider에 `autoDispose`를 붙였다. 메모리 누수를 줄인다는 생각이었다. 그런데 Android에서 화면 회전 후 연결 상태가 `disconnected -> connecting -> connected`로 다시 튀었다. 실제 연결은 살아 있는데 UI 이벤트만 다시 발생한 것이다. 그 뒤로 연결 세션과 화면 표시 Provider를 분리했다.

## 짧게 남기는 기준

- Flutter BLE 재연결 상태는 Widget에서 직접 listen하지 않는다.
- `StreamNotifier`는 BLE 연결 Stream과 수동 reconnect 명령이 같이 필요할 때 쓴다.
- `flutter_blue_plus` 객체는 Repository 아래에 숨기고, Provider에는 앱 전용 상태만 흘린다.
- `autoDispose`는 등록·상세 화면처럼 짧은 수명에만 붙인다.
- 홈 대시보드는 연결 세션을 직접 소유하지 않고, 화면 모델만 구독한다.

솔직하게 정리하면 `StreamNotifier`가 모든 BLE 문제를 해결하지는 않는다. iOS 시뮬레이터에서는 BLE 테스트 자체가 안 되고, Android 제조사별 백그라운드 정책도 다르다. 그래도 연결 이벤트가 흩어지는 문제는 꽤 줄었다. Flutter IoT 앱에서 BLE 상태를 Riverpod으로 옮길 때는 “어디서 listen할까”보다 “이 연결은 화면 수명인가, 앱 세션 수명인가”를 정하는 게 더 중요했다.
