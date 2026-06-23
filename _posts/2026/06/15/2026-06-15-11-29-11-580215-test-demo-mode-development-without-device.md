---
layout: post
title: "테스트/데모 모드 구현 - 실제 기기 없이도 개발하기"
description: " "
date: 2026-06-15
tags: [Flutter, 테스트모드, Mock, 데모, 개발환경]
comments: true
share: true
---

# 테스트/데모 모드 구현 - 실제 기기 없이도 개발하기

![개발 테스트](https://images.unsplash.com/photo-1614064641938-3bbee52942c7?w=800&q=80)

## 실제 IoT 기기 없이 개발해야 하는 현실

IoT 앱 개발의 가장 큰 어려움은 개발 환경이다. 코드를 짜면서 실제 IoT 기기에 연결해서 테스트하는 건 현실적으로 힘들다. IoT 기기가 있더라도 항상 켜두기 어렵고, 팀원마다 기기가 있는 것도 아니다.

이 문제를 해결하기 위해 테스트/데모 모드를 구현했다.

## 구조

`data/test/` 디렉터리에 각 Repository의 테스트 구현체가 있다:

```
lib/data/test/
├── test_api_impl.dart         # API 요청 Mock
├── test_ble_impl.dart         # BLE 연결 Mock
├── test_iot_device_impl.dart      # IoT 기기 MQTT Mock
├── test_iot_device_json.dart      # 테스트용 JSON 데이터
├── test_db_impl.dart          # DB Mock
├── test_iot_impl.dart         # IoT 서비스 Mock
├── test_iot_dashboard_impl.dart       # IoT 모니터링 대시보드 Mock
└── test_wifi_only_impl.dart   # WiFi-only 기기 Mock
```

## TestIoT DeviceMqttRepositoryImpl 예시

```dart
// data/test/test_iot_device_impl.dart
class TestIoT DeviceMqttRepositoryImpl implements IoT DeviceMqttRepository {
  final _statusController = StreamController<IoT DeviceStatus>.broadcast();
  
  // 초기 상태
  IoT DeviceStatus _currentStatus = const IoT DeviceStatus(
    deviceId: 'TEST_DEVICE_001',
    isOn: false,
    currentTemp: 20.0,
    targetTemp: 22.0,
    mode: IoT DeviceMode.heating,
  );

  @override
  Stream<IoT DeviceStatus> get statusStream => _statusController.stream;

  @override
  Future<void> publishPowerControl(String deviceId, bool isOn) async {
    // 실제 MQTT 대신 로컬에서 상태 변경
    await Future.delayed(const Duration(milliseconds: 500)); // 네트워크 지연 시뮬레이션
    
    _currentStatus = _currentStatus.copyWith(isOn: isOn);
    _statusController.add(_currentStatus);
  }

  @override
  Future<void> publishTemperatureControl(String deviceId, int temperature) async {
    await Future.delayed(const Duration(milliseconds: 300));
    
    _currentStatus = _currentStatus.copyWith(targetTemp: temperature.toDouble());
    _statusController.add(_currentStatus);
    
    // 현재 온도가 목표 온도에 도달하는 시뮬레이션
    _simulateTemperatureChange(temperature.toDouble());
  }

  void _simulateTemperatureChange(double target) {
    Timer.periodic(const Duration(seconds: 2), (timer) {
      final current = _currentStatus.currentTemp;
      final diff = target - current;
      
      if (diff.abs() < 0.1) {
        timer.cancel();
        return;
      }
      
      final newTemp = current + (diff > 0 ? 0.5 : -0.5);
      _currentStatus = _currentStatus.copyWith(currentTemp: newTemp);
      _statusController.add(_currentStatus);
    });
  }

  @override
  void dispose() {
    _statusController.close();
  }
}
```

## GetX Binding에서 환경별 교체

```dart
// core/bindings/initial_binding.dart
class InitialBinding extends Bindings {
  final Config config;
  InitialBinding(this.config);

  @override
  void dependencies() {
    if (config.isTestMode) {
      // 테스트 구현체 등록
      Get.lazyPut<IoT DeviceMqttRepository>(
        () => TestIoT DeviceMqttRepositoryImpl(),
        fenix: true,
      );
      Get.lazyPut<ApiRepository>(
        () => TestApiRepositoryImpl(),
        fenix: true,
      );
      Get.lazyPut<BleRepository>(
        () => TestBleRepositoryImpl(),
        fenix: true,
      );
    } else {
      // 실제 구현체 등록
      Get.lazyPut<IoT DeviceMqttRepository>(
        () => IoT DeviceMqttRepositoryImpl(Get.find()),
        fenix: true,
      );
      // ...
    }
  }
}
```

## 테스트 JSON 데이터

`test_iot_device_json.dart`에 테스트용 응답 JSON을 하드코딩해둔다:

```dart
// data/test/test_iot_device_json.dart
class TestIoT DeviceJson {
  static const deviceList = '''
  [
    {
      "device_id": "TEST001",
      "nick_name": "거실 IoT 기기",
      "model_name": "NPE-40KW",
      "is_online": true
    },
    {
      "device_id": "TEST002", 
      "nick_name": "안방 IoT 기기",
      "model_name": "NPE-25KW",
      "is_online": false
    }
  ]
  ''';
}
```

## 데모 모드 vs 개발 모드

두 가지 "Mock 모드"가 있다:

**개발 모드 (테스트 구현체)**:
- `--dart-define=CONFIG=dev` 환경에서 동작
- 실제 API 서버는 연결되고, 기기 제어만 Mock
- 개발자가 기기 없이 UI를 개발할 때 사용

**데모 모드 (앱 심사용)**:
- Firebase Remote Config로 활성화
- 실제 서버가 아닌 Stage 서버로 연결
- 심사자가 가상의 계정으로 기능 테스트 가능

---

다음 편은 멀티 환경 빌드 설정이다.
