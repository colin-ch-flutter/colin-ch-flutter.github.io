---
layout: post
title: "BLE 연결 안정성 높이기 - 재연결과 오류 처리 전략"
description: " "
date: 2026-05-11
tags: [Flutter, BLE, 재연결, 오류처리, flutter_blue_plus]
comments: true
share: true
---

# BLE 연결 안정성 높이기 - 재연결과 오류 처리 전략

![회로 연결](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## BLE가 끊기는 상황들

BLE 연결은 생각보다 불안정하다. 프로비저닝 과정은 짧게는 30초, 길게는 1-2분이 걸리는데, 이 시간 동안 연결이 끊길 수 있는 원인이 여럿 있다:

- **거리**: 스마트폰을 IoT 기기에서 너무 멀리 들고 가는 경우
- **전파 간섭**: 전자레인지, 다른 WiFi 기기의 간섭
- **iOS 백그라운드 제한**: 앱이 백그라운드로 가면 BLE 연결이 끊길 수 있음
- **기기 측 타임아웃**: ESP32에서 일정 시간 통신이 없으면 연결을 끊는 경우
- **Android 5.0 이하 버그**: 구형 Android에서 BLE 스택 버그

## 재연결 로직

지수 백오프(exponential backoff) 방식으로 재연결을 시도한다. 처음에는 빠르게 재시도하고, 실패가 반복될수록 간격을 늘린다:

```dart
class BleReconnector {
  static const maxRetries = 3;
  static const baseDelay = Duration(seconds: 1);

  int _retryCount = 0;

  Future<bool> reconnect(BluetoothDevice device) async {
    while (_retryCount < maxRetries) {
      try {
        final delay = baseDelay * (1 << _retryCount); // 1s, 2s, 4s
        await Future.delayed(delay);

        await device.connect(timeout: const Duration(seconds: 10));
        _retryCount = 0;
        return true;
      } catch (e) {
        _retryCount++;
        print('재연결 시도 $_retryCount/$maxRetries 실패: $e');
      }
    }
    return false;
  }
}
```

## 연결 상태 모니터링

연결이 끊겼을 때 즉시 감지하기 위해 상태 스트림을 구독한다:

```dart
device.connectionState.listen((state) {
  if (state == BluetoothConnectionState.disconnected) {
    _handleUnexpectedDisconnect();
  }
});

void _handleUnexpectedDisconnect() {
  // 프로비저닝 중에 끊긴 경우
  if (_isProvisioning) {
    _showReconnectDialog();
    _reconnector.reconnect(_device!).then((success) {
      if (success) {
        // 끊기기 전 단계부터 재개
        _resumeProvisioning();
      } else {
        _showFailureDialog();
      }
    });
  }
}
```

## Android vs iOS 차이

BLE 처리에서 Android와 iOS 사이에 가장 크게 체감한 차이는 이것들이다.

**연결 속도**: Android가 iOS보다 연결이 느린 경우가 있다. 특히 구형 Android에서는 `discoverServices()`가 5-10초씩 걸리기도 한다.

**자동 재연결**: iOS는 시스템 수준에서 BLE 연결을 유지하려는 경향이 있다. Android는 앱이 명시적으로 재연결을 시도해야 한다.

**권한 처리**:
```dart
// Android 12+에서는 BLUETOOTH_CONNECT 권한도 필요
if (Platform.isAndroid) {
  final androidInfo = await DeviceInfoPlugin().androidInfo;
  if (androidInfo.version.sdkInt >= 31) {
    // BLUETOOTH_SCAN + BLUETOOTH_CONNECT 권한 요청
  } else {
    // ACCESS_FINE_LOCATION 권한 요청
  }
}
```

## 사용자에게 적절한 피드백 보여주기

기술적인 에러 메시지를 그대로 보여주면 안 된다. 사용자가 이해할 수 있는 언어로 번역해야 한다:

```dart
String bleErrorToUserMessage(dynamic error) {
  final msg = error.toString().toLowerCase();
  
  if (msg.contains('timeout')) {
    return '연결 시간이 초과됐습니다. 스마트폰을 IoT 기기 가까이에서 다시 시도해주세요.';
  }
  if (msg.contains('not found') || msg.contains('device not found')) {
    return '기기를 찾을 수 없습니다. IoT 기기 전원이 켜져 있는지 확인해주세요.';
  }
  if (msg.contains('permission')) {
    return '블루투스 권한이 필요합니다.';
  }
  return '연결 중 오류가 발생했습니다. 다시 시도해주세요.';
}
```

## 실제 QA에서 발견한 버그들

QA를 하면서 발견한 실제 BLE 관련 버그들을 몇 가지 공유한다.

**Samsung 갤럭시에서 간헐적 연결 실패**: 특정 삼성 기기에서 `discoverServices()`가 빈 목록을 반환하는 경우가 있었다. 연결 직후 약간의 딜레이를 주는 것으로 해결했다.

```dart
await device.connect();
await Future.delayed(const Duration(milliseconds: 500)); // 안정화 대기
await device.discoverServices();
```

**iOS에서 앱 재시작 후 연결 불가**: 이전 연결이 완전히 해제되지 않은 상태로 앱이 재시작되면 연결 시도가 즉시 실패한다. 연결 전에 기존 연결을 명시적으로 끊는 것으로 해결했다.

---

다음 편부터는 MQTT 시리즈가 시작된다.
