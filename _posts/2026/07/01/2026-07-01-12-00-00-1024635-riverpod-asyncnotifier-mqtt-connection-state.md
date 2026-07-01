---
layout: post
title: "Flutter IoT MQTT 연결 상태 - Riverpod AsyncNotifier로 GetX Controller 줄이기"
description: "Flutter IoT 앱에서 MQTT 연결 끊김 상태를 Riverpod AsyncNotifier로 관리하며 GetX Controller 비대화를 줄인 과정을 정리했다."
date: 2026-07-01
tags: [Flutter, GetX, MQTT, IoT, 스마트홈]
comments: true
share: true
---
![Flutter IoT MQTT 연결 상태 관리](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Flutter IoT 앱에서 MQTT 연결 끊김을 다루려면 화면 Controller가 아니라 연결 상태 자체를 독립된 상태로 빼는 편이 낫다. Flutter 스마트홈 앱을 GetX로 만들 때는 `DeviceController` 안에 MQTT 연결, 토픽 구독, 재연결 버튼 상태까지 들어갔다. 해보니 됐지만, 앱이 커질수록 어디서 연결을 끊었는지 추적하기가 점점 어려워졌다.

[MQTT 기초]({% post_url 2026-05-12-10-31-49-160485-mqtt-protocol-fundamentals-iot-heart %})를 정리할 때도 느꼈지만, MQTT는 연결 한 번으로 끝나는 구조가 아니다. 앱 포그라운드 복귀, 네트워크 전환, 토큰 갱신, 브로커 keep alive 실패가 모두 연결 상태를 흔든다. 그래서 Riverpod으로 옮길 때 `AsyncNotifier`를 MQTT 세션 상태의 중심으로 뒀다.

## GetX Controller가 비대해지는 지점

처음 구조는 대충 이랬다.

```dart
class DeviceController extends GetxController {
  final isConnected = false.obs;
  final isReconnecting = false.obs;
  final devices = <Device>[].obs;

  Future<void> connectMqtt() async {}
  Future<void> reconnectMqtt() async {}
  Future<void> subscribeDeviceTopics() async {}
  Future<void> togglePower(String deviceId) async {}
}
```

문제는 `togglePower()`가 실패했을 때였다. 기기 명령 실패인지, MQTT 연결 끊김인지, 토픽 구독 누락인지 화면에서 구분하기가 애매했다. 결국 스낵바 문구도 `제어에 실패했습니다` 같은 말로 뭉개졌다. 실제 사용자는 와이파이가 잠깐 바뀐 상황인데 앱은 기기 고장처럼 보였다.

## AsyncNotifier로 연결 상태를 따로 둔다

Riverpod에서는 MQTT 연결을 화면 상태가 아니라 앱 인프라 상태로 본다. Repository는 실제 연결을 담당하고, `AsyncNotifier`는 UI가 읽기 좋은 상태로 변환한다.

```dart
final mqttSessionProvider =
    AsyncNotifierProvider<MqttSessionNotifier, MqttSessionState>(
  MqttSessionNotifier.new,
);

class MqttSessionNotifier extends AsyncNotifier<MqttSessionState> {
  late final MqttRepository _repository;

  @override
  Future<MqttSessionState> build() async {
    _repository = ref.read(mqttRepositoryProvider);
    final connected = await _repository.connect();

    return MqttSessionState(
      connected: connected,
      reconnecting: false,
    );
  }

  Future<void> reconnect() async {
    state = const AsyncLoading();

    state = await AsyncValue.guard(() async {
      final connected = await _repository.reconnect();
      return MqttSessionState(
        connected: connected,
        reconnecting: false,
      );
    });
  }
}
```

여기서 마음에 들었던 점은 실패 상태가 `AsyncError`로 남는다는 것이다. GetX에서는 `isConnected`, `errorMessage`, `isReconnecting`을 따로 맞춰야 했다. 하나라도 갱신 순서가 꼬이면 화면은 연결됨으로 보이는데 버튼은 비활성화되는 이상한 상태가 나왔다.

## 화면은 상태를 읽고 명령만 보낸다

제어 화면은 MQTT 연결 여부를 직접 계산하지 않는다. `ref.watch()`로 세션을 보고, 연결이 없으면 명령 버튼을 잠근다.

```dart
class DevicePowerButton extends ConsumerWidget {
  const DevicePowerButton({super.key, required this.deviceId});

  final String deviceId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final mqttSession = ref.watch(mqttSessionProvider);

    final enabled = mqttSession.maybeWhen(
      data: (session) => session.connected,
      orElse: () => false,
    );

    return ElevatedButton(
      onPressed: enabled
          ? () => ref.read(deviceCommandProvider).togglePower(deviceId)
          : null,
      child: const Text('전원'),
    );
  }
}
```

솔직하게 정리해보면, 이 구조는 코드 줄 수를 갑자기 줄여주진 않는다. 대신 실패 위치가 선명해진다. [AWS IoT 연결]({% post_url 2026-05-13-11-38-02-172830-aws-iot-core-connection-sigv4-authentication %})에서 겪었던 것처럼 인증 서명이 틀리면 서버는 `403 Forbidden`만 던진다. 이 에러가 화면 Controller 안에서 섞이면 버튼 로직까지 같이 의심하게 된다.

## 주의할 점

`build()` 안에서 무조건 `connect()`를 호출하면 provider가 dispose됐다가 다시 생성될 때 재연결이 반복될 수 있다. IoT 앱에서는 화면 이동만으로 MQTT 세션이 흔들리면 안 된다. 실제 앱에서는 `keepAlive` 설정을 두거나, 앱 생명주기와 세션 생명주기를 분리해서 관리하는 편이 낫다.

또 하나는 자동 재시도 욕심이다. 재연결을 너무 빠르게 반복하면 모바일 네트워크가 불안정한 상황에서 배터리와 서버 로그를 같이 태운다. 나는 1초, 2초, 5초 정도로 제한하고, 사용자가 직접 재시도할 수 있는 버튼을 남기는 쪽이 실전에서 더 낫다고 봤다.

## 짧은 정리

- Flutter IoT의 MQTT 연결 상태는 화면 Controller보다 별도 세션 상태로 분리하는 편이 낫다.
- Riverpod `AsyncNotifier`를 쓰면 연결 중, 성공, 실패 상태를 UI가 일관되게 읽을 수 있다.
- MQTT 연결 끊김은 기기 제어 실패와 섞지 말고, 사용자에게 네트워크 문제로 구분해 보여줘야 한다.
- 재연결은 자동화하되 너무 공격적으로 반복하지 않는 게 스마트홈 앱 운영에 맞다.
