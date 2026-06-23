---
layout: post
title: "mqtt5_client로 IoT 기기 On/Off 제어하기"
description: " "
date: 2026-05-15
tags: [Flutter, MQTT5, IoT 기기제어, IoT, Dart]
comments: true
share: true
---

# mqtt5_client로 IoT 기기 On/Off 제어하기

![스마트홈 컨트롤](https://images.unsplash.com/photo-1585771724684-38269d6639fd?w=800&q=80)

## mqtt5_client vs mqtt_client

프로젝트에 두 패키지가 모두 있다. `mqtt_client`는 MQTT3.1.1, `mqtt5_client`는 MQTT5를 지원한다.

AWS IoT Core가 MQTT5를 지원하면서 MQTT5의 확장 기능(이유 코드, 사용자 속성 등)을 활용할 수 있게 됐다. 신규 기기 연동은 mqtt5_client로 하고, 레거시 연동은 기존 mqtt_client를 유지했다.

## 연결 설정

```dart
// data/services/iot_service.dart
class IotMqttService {
  late MqttServerClient _client;
  
  Future<void> connect({
    required String endpoint,
    required String clientId,
    required String websocketUrl,  // SigV4 서명된 URL
  }) async {
    _client = MqttServerClient.withPort(endpoint, clientId, 443);
    _client.secure = true;
    _client.useWebSocket = true;
    _client.websocketProtocols = MqttClientConstants.protocolsSingleDefault;
    _client.keepAlivePeriod = 60;
    _client.autoReconnect = true;
    
    final connMessage = MqttConnectMessage()
        .withClientIdentifier(clientId)
        .startClean()
        .withWillQos(MqttQos.atLeastOnce);
    
    _client.connectionMessage = connMessage;
    
    _client.onConnected = _onConnected;
    _client.onDisconnected = _onDisconnected;
    _client.onAutoReconnect = _onAutoReconnect;
    
    await _client.connect();
  }
  
  void _onConnected() => print('MQTT 연결됨');
  void _onDisconnected() => print('MQTT 연결 끊김');
  void _onAutoReconnect() => print('MQTT 자동 재연결 시도');
}
```

## 토픽 구독

```dart
void subscribe(String topic, {MqttQos qos = MqttQos.atLeastOnce}) {
  _client.subscribe(topic, qos);
}

// 메시지 수신 스트림
Stream<MqttReceivedMessage<MqttMessage?>> get messageStream {
  return _client.updates!;
}
```

실제 사용:

```dart
_iotService.subscribe('smarthome/device/$deviceId/status');
_iotService.subscribe('smarthome/device/$deviceId/event');

_iotService.messageStream.listen((messages) {
  for (final msg in messages) {
    final topic = msg.topic;
    final payload = MqttPublishPayload.bytesToStringAsString(
      (msg.payload as MqttPublishMessage).payload.message,
    );
    _handleMessage(topic, payload);
  }
});
```

## IoT 기기 전원 제어 발행

```dart
Future<void> publishPowerControl(String deviceId, bool isOn) async {
  final topic = 'smarthome/device/$deviceId/command/power';
  final payload = jsonEncode({
    'action': isOn ? 'on' : 'off',
    'timestamp': DateTime.now().millisecondsSinceEpoch,
    'requestId': const Uuid().v4(),  // 응답 매칭용
  });
  
  final builder = MqttClientPayloadBuilder();
  builder.addString(payload);
  
  _client.publishMessage(
    topic,
    MqttQos.atLeastOnce,
    builder.payload!,
    retain: false,
  );
}
```

## 온도 설정 발행

```dart
Future<void> publishTemperatureControl(String deviceId, int temperature) async {
  // 온도 범위 검증
  if (temperature < 10 || temperature > 80) {
    throw ArgumentError('온도는 10~80 사이여야 합니다: $temperature');
  }
  
  final topic = 'smarthome/device/$deviceId/command/temperature';
  final payload = jsonEncode({
    'temperature': temperature,
    'unit': 'celsius',
  });
  
  final builder = MqttClientPayloadBuilder();
  builder.addString(payload);
  
  _client.publishMessage(topic, MqttQos.atLeastOnce, builder.payload!);
}
```

## 응답 대기

명령을 보낸 후 기기의 응답을 기다리는 패턴:

```dart
Future<IoT DeviceStatus> sendCommandAndWait(
  String deviceId,
  Map<String, dynamic> command, {
  Duration timeout = const Duration(seconds: 10),
}) async {
  final requestId = const Uuid().v4();
  final completer = Completer<IoT DeviceStatus>();
  
  // 응답 대기 등록
  _pendingRequests[requestId] = completer;
  
  // 명령 발행
  await publishCommand(deviceId, {...command, 'requestId': requestId});
  
  // 타임아웃과 함께 대기
  return completer.future.timeout(
    timeout,
    onTimeout: () {
      _pendingRequests.remove(requestId);
      throw TimeoutException('기기 응답 대기 시간 초과');
    },
  );
}

void _handleMessage(String topic, String payload) {
  final data = jsonDecode(payload);
  final requestId = data['requestId'] as String?;
  
  if (requestId != null && _pendingRequests.containsKey(requestId)) {
    final completer = _pendingRequests.remove(requestId)!;
    completer.complete(IoT DeviceStatus.fromJson(data));
  }
  
  // 스트림에도 발행 (전체 상태 업데이트)
  _statusController.add(IoT DeviceStatus.fromJson(data));
}
```

## 실제로 IoT 기기가 응답했을 때의 감동

개발하다 보면 가끔 "이게 진짜 동작하네"라는 순간이 온다. 에뮬레이터에서 On 버튼을 누르자 실제 IoT 기기 불이 켜지는 걸 처음 봤을 때가 그랬다. 앱에서 MQTT로 명령을 보내면 AWS IoT Core를 거쳐서 IoT 기기까지 전달되는데, 그 전체 여정이 1-2초 안에 이루어진다.

---

다음 편은 MQTT 메시지를 받아서 UI를 실시간으로 업데이트하는 방법이다.
