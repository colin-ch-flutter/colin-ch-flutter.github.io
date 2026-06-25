---
layout: post
title: "MQTT 연결 유지하기 - Keep-Alive, Will, Reconnect 전략"
description: " "
date: 2026-05-17
tags: [Flutter, MQTT, 연결관리, KeepAlive, 재연결]
comments: true
share: true
---
![네트워크 안정성](https://images.unsplash.com/photo-1544197150-b99a580bb7a8?w=800&q=80)

## MQTT 연결이 끊기는 상황들

IoT 앱은 백그라운드에서도 기기 상태 알림을 받아야 한다. 그런데 MQTT 연결은 다양한 이유로 끊긴다:

- 스마트폰 화면이 꺼지고 앱이 백그라운드로 이동
- WiFi에서 LTE로 네트워크 전환
- 일시적인 인터넷 불안정
- 서버 점검이나 재시작
- Keep-Alive 타임아웃

이런 상황에서 연결이 자동으로 복구되어야 한다.

## Keep-Alive

MQTT Keep-Alive는 클라이언트가 일정 간격으로 브로커에 "나 살아있어"(PINGREQ) 신호를 보내는 메커니즘이다. 브로커가 응답(PINGRESP)하지 않으면 연결이 끊겼다고 판단한다.

```dart
final connMessage = MqttConnectMessage()
    .withClientIdentifier(clientId)
    .keepAliveFor(60);  // 60초 간격으로 ping
```

60초로 설정하면 60초 안에 서로 통신이 없으면 ping을 보낸다. AWS IoT Core의 기본 Keep-Alive 제한이 1200초이니 60초는 충분히 안전하다.

## 자동 재연결 설정

`mqtt5_client`는 자동 재연결을 지원한다:

```dart
_client.autoReconnect = true;
_client.resubscribeOnAutoReconnect = true; // 재연결 후 토픽 재구독

_client.onAutoReconnect = () {
  print('MQTT 자동 재연결 시도 중...');
};

_client.onAutoReconnected = () {
  print('MQTT 자동 재연결 성공');
  // 재연결 후 처리가 필요한 작업
  _resubscribeAllTopics();
};
```

## 앱 생명주기에 따른 MQTT 관리

앱이 백그라운드로 가거나 다시 포그라운드로 올 때 MQTT를 관리해야 한다:

```dart
// presentation/main/main_tab_controller.dart
class MainTabController extends GetxController with WidgetsBindingObserver {
  @override
  void onInit() {
    super.onInit();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:
        // 포그라운드로 돌아왔을 때
        _reconnectIfNeeded();
        break;
      case AppLifecycleState.paused:
        // 백그라운드로 갔을 때
        // Android: MQTT 유지 (백그라운드 서비스 없으면 끊길 수 있음)
        // iOS: 백그라운드에서 MQTT 유지 어려움
        break;
      default:
        break;
    }
  }

  void _reconnectIfNeeded() {
    if (!_mqttService.isConnected) {
      _mqttService.reconnect();
    }
  }

  @override
  void onClose() {
    WidgetsBinding.instance.removeObserver(this);
    super.onClose();
  }
}
```

## 네트워크 상태 감지

`connectivity_plus`로 네트워크 변경을 감지한다:

```dart
// core/services/auth_manager.dart 에서 활용
Connectivity().onConnectivityChanged.listen((result) {
  final isConnected = result != ConnectivityResult.none;
  
  if (isConnected && !_mqttService.isConnected) {
    // 네트워크 복구 시 MQTT 재연결
    _mqttService.reconnect();
  }
});
```

네트워크가 복구됐다고 해서 즉시 연결을 시도하면 실패할 수 있다. 약간의 딜레이를 주는 게 안전하다:

```dart
if (isConnected && !_mqttService.isConnected) {
  await Future.delayed(const Duration(seconds: 2));
  await _mqttService.reconnect();
}
```

## Last Will Testament (LWT)

앱이 예기치 않게 종료되면 브로커가 LWT 메시지를 발행한다:

```dart
final connMessage = MqttConnectMessage()
    .withClientIdentifier(clientId)
    .withWillTopic('smarthome/app/$userId/presence')
    .withWillMessage(jsonEncode({
      'status': 'offline',
      'timestamp': DateTime.now().millisecondsSinceEpoch,
    }))
    .withWillQos(MqttQos.atLeastOnce)
    .withWillRetain();
```

정상 종료 시에는 직접 offline 메시지를 발행하고 연결을 끊는다:

```dart
Future<void> disconnect() async {
  // 정상 종료 메시지
  await publish(
    'smarthome/app/$userId/presence',
    jsonEncode({'status': 'offline'}),
    retain: true,
  );
  
  _client.disconnect();
}
```

## 실제로 겪은 문제

iOS에서 앱이 백그라운드로 가면 MQTT 연결이 끊기는 문제가 있었다. iOS는 백그라운드 네트워크 연결을 적극적으로 제한하기 때문이다.

대안으로 FCM 푸시를 기본으로 쓰고, 앱이 포그라운드에 있을 때만 MQTT 연결을 유지하는 방식을 채택했다. 앱이 포그라운드에 있을 때는 MQTT로 실시간 상태를 받고, 백그라운드에서는 FCM으로 중요 알림만 받는 구조다.

---

다음 편부터는 데이터 레이어 파트다. Realm DB부터 시작한다.
