---
layout: post
title: "Flutter IoT Riverpod family - 기기별 상태를 deviceId로 분리하기"
description: "Flutter IoT 앱에서 Riverpod family로 Flutter 스마트홈 기기별 MQTT 상태와 Flutter BLE 연결 상태를 deviceId 기준으로 분리한 방법을 정리했다."
date: 2026-07-01
tags: [Flutter, Dart, BLE, MQTT, IoT]
comments: true
share: true
---
![Flutter IoT Riverpod family](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

Flutter IoT 앱에서 Flutter 스마트홈 기기 상태를 Riverpod으로 옮길 때 `family`는 deviceId별 상태 분리에 가장 잘 맞았다. MQTT 상태, Flutter BLE 연결 상태, 보일러 전원 값을 하나의 Controller나 Provider에 몰아넣으면 기기가 1대일 때는 편한데, 3대만 넘어가도 어떤 상태가 어떤 기기 것인지 계속 헷갈린다.

처음엔 `Map<String, DeviceState>` 하나로 충분하다고 봤다. GetX Controller에서 쓰던 방식과 비슷했고, MQTT 메시지가 들어오면 `state[deviceId] = newState`처럼 갱신하면 됐다. 문제는 화면이었다. 거실 보일러 카드 하나의 온도만 바뀌었는데 침실, 서재 카드까지 같이 리빌드됐다. 로그를 찍어보니 MQTT 패킷 하나에 대시보드 전체가 흔들리고 있었다.

## Map 하나로 모든 기기를 들고 있으면 생기는 문제

기기 목록 화면을 이렇게 만들면 시작은 쉽다.

```dart
final deviceStatesProvider =
    NotifierProvider<DeviceStatesNotifier, Map<String, DeviceState>>(
  DeviceStatesNotifier.new,
);
```

하지만 `ref.watch(deviceStatesProvider)`를 카드마다 호출하면 모든 카드가 같은 Provider를 본다. 특정 deviceId만 꺼내 쓰더라도 원본 Map이 바뀌면 구독자는 전부 다시 build된다. 성능보다 더 귀찮았던 건 조건문이었다. BLE 재연결 중인 기기와 MQTT 온라인 기기가 섞이면 `isConnecting`, `mqttOnline`, `lastSeen`이 한 Map 안에서 같이 움직인다.

## family로 deviceId를 Provider의 인자로 만든다

기기 상세 상태는 deviceId를 기준으로 잘라내는 편이 낫다.

```dart
final deviceStatusProvider =
    StreamProvider.family<DeviceStatus, String>((ref, deviceId) {
  final repository = ref.watch(deviceRepositoryProvider);

  return repository.watchStatus(deviceId);
});
```

화면에서는 카드가 자기 deviceId만 본다.

```dart
class DeviceCard extends ConsumerWidget {
  final String deviceId;
  const DeviceCard({super.key, required this.deviceId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final status = ref.watch(deviceStatusProvider(deviceId));

    return status.when(
      data: (value) => Text('${value.temperature}도'),
      loading: () => const Text('연결 확인 중'),
      error: (_, __) => const Text('상태 없음'),
    );
  }
}
```

이렇게 바꾸면 거실 보일러 상태가 바뀌어도 `deviceStatusProvider('living-room')`만 갱신된다. 침실 카드는 `deviceStatusProvider('bedroom')`을 보고 있으니 불필요하게 다시 그릴 이유가 줄어든다.

## BLE와 MQTT 상태는 같은 family에 억지로 넣지 않는다

처음 실수한 부분은 BLE 연결 상태와 MQTT 상태를 하나의 `DeviceStatus`에 다 넣은 것이다. 실제로는 수명이 다르다. BLE는 등록이나 근거리 제어 중에만 살아 있고, MQTT는 앱 실행 내내 유지된다. 그래서 Provider도 분리했다.

```dart
final bleConnectionProvider =
    StreamProvider.family<BleConnectionState, String>((ref, deviceId) {
  return ref.watch(bleRepositoryProvider).watchConnection(deviceId);
});

final mqttStatusProvider =
    StreamProvider.family<MqttDeviceStatus, String>((ref, deviceId) {
  return ref.watch(mqttRepositoryProvider).watchDevice(deviceId);
});
```

카드에서는 필요한 것만 조합한다. 전원 버튼은 MQTT만 보면 되고, 등록 배너는 BLE만 보면 된다. 둘을 합친 값이 필요할 때만 얇은 Provider를 둔다.

```dart
final deviceCardProvider =
    Provider.family<DeviceCardState, String>((ref, deviceId) {
  final mqtt = ref.watch(mqttStatusProvider(deviceId)).valueOrNull;
  final ble = ref.watch(bleConnectionProvider(deviceId)).valueOrNull;

  return DeviceCardState(
    online: mqtt?.online ?? false,
    bleConnecting: ble == BleConnectionState.connecting,
    temperature: mqtt?.temperature,
  );
});
```

## 주의할 점

`family`를 쓰면 deviceId가 Provider 캐시 키가 된다. 그래서 `deviceId`는 화면마다 같은 값을 넘겨야 한다. 별칭, MAC 주소, 서버 ID를 섞어 쓰면 같은 기기를 두 Provider로 착각한다. 나는 등록 화면에서는 BLE MAC을 쓰고 상세 화면에서는 서버 deviceId를 써서 상태가 비는 버그를 만들었다.

또 하나는 수명 관리다. 기기 상세 화면처럼 잠깐 보는 상태라면 `autoDispose`를 붙이는 편이 낫다. 반대로 홈 대시보드처럼 자주 돌아오는 화면은 캐시가 너무 빨리 사라지면 로딩이 반복된다.

짧게 정리하면 이렇다. Flutter IoT에서 Riverpod family는 기기별 상태를 deviceId로 분리할 때 효과가 크다. MQTT와 Flutter BLE 상태는 수명이 다르니 Provider를 나누고, UI에서 필요한 조합만 만든다. Map 하나로 모든 기기를 관리하는 방식은 빨리 만들 수 있지만, 스마트홈 앱이 커질수록 리빌드와 테스트가 같이 무거워진다.
