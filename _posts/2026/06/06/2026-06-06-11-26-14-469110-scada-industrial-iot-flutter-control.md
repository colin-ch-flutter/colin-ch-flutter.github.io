---
layout: post
title: "IoT 모니터링 대시보드란? 산업용 IoT를 Flutter 앱에서 제어하기"
description: " "
date: 2026-06-06
tags: [Flutter, IoT 모니터링 대시보드, 산업IoT, IoT 기기, 원격제어]
comments: true
share: true
---

# IoT 모니터링 대시보드란? 산업용 IoT를 Flutter 앱에서 제어하기

![산업 제어 시스템](https://images.unsplash.com/photo-1581091226825-a6a2a5aee158?w=800&q=80)

## IoT 모니터링 대시보드가 뭔가

IoT 모니터링 대시보드는 Supervisory Control and Data Acquisition의 약자다. 번역하면 "감시 제어 및 데이터 수집"이다.

공장, 발전소, 빌딩 같은 대형 시설의 장비를 원격으로 감시하고 제어하는 시스템이다. 기존에는 전용 HMI(Human-Machine Interface) 패널이나 PC 소프트웨어로만 제어했는데, 스마트홈은 이걸 스마트폰 앱으로도 할 수 있게 했다.

## 일반 IoT 기기와 IoT 모니터링 대시보드 IoT 기기의 차이

일반 가정용 IoT 기기와 IoT 모니터링 대시보드 IoT 기기는 규모와 복잡도가 다르다:

| 구분 | 일반 IoT 기기 | IoT 모니터링 대시보드 IoT 기기 |
|------|-------------|--------------|
| 용도 | 단독주택, 아파트 | 빌딩, 공장, 복합시설 |
| 제어 포인트 | 온도, 전원, 모드 | 수십 개 이상의 파라미터 |
| 연결 방식 | 직접 WiFi | 전용 게이트웨이 경유 |
| 알람 | 간단한 오류 코드 | 상세한 알람 목록 |
| 히스토리 | 기본적인 사용 기록 | 상세 운전 이력, EMS |

## IoT 모니터링 대시보드 시스템 구조

```
[IoT 기기 기기들]
    ↓ RS485/Modbus
[IoT 모니터링 대시보드 게이트웨이]
    ↓ MQTT (AWS IoT Core)
[클라우드 서버]
    ↓ API + MQTT
[스마트홈 앱]
```

일반 IoT 기기는 기기가 직접 WiFi로 AWS IoT Core에 연결되지만, IoT 모니터링 대시보드 IoT 기기는 게이트웨이를 거친다. 게이트웨이가 여러 IoT 기기를 묶어서 클라우드에 연결한다.

## MQTT 프로토콜 차이

IoT 모니터링 대시보드와 일반 IoT 기기의 MQTT 메시지 구조가 다르다. 더 많은 데이터 포인트를 포함한다:

```json
// 일반 기기 상태
{
  "power": "on",
  "mode": "heating",
  "currentTemp": 22.5,
  "targetTemp": 24
}

// IoT 모니터링 대시보드 기기 상태 (훨씬 복잡)
{
  "systemStatus": 1,
  "zones": [
    {"id": 1, "name": "1층 난방", "temp": 22.5, "setpoint": 24, "active": true},
    {"id": 2, "name": "2층 난방", "temp": 20.1, "setpoint": 22, "active": false}
  ],
  "alarms": [
    {"code": "E001", "severity": "warning", "message": "센서 이상"}
  ],
  "energyData": {
    "gasUsage": 1.2,
    "powerUsage": 0.5,
    "efficiency": 92.5
  }
}
```

## Repository 분리

IoT 모니터링 대시보드와 일반 IoT 기기의 Repository를 분리했다:

```
domain/repositories/
├── iot_device_mqtt_repository.dart       # 일반 IoT 기기
└── iot_dashboard_mqtt_repository.dart        # IoT 모니터링 대시보드 IoT 기기

data/repositories/
├── iot_device_mqtt_repository_impl.dart
└── iot_dashboard_mqtt_repository_impl.dart
```

프로토콜이 다르기 때문에 Repository 인터페이스 자체가 다르다:

```dart
// domain/repositories/iot_dashboard_mqtt_repository.dart
abstract class ScadaMqttRepository {
  Stream<ScadaStatus> get statusStream;
  Stream<List<ScadaAlarm>> get alarmStream;
  
  Future<void> controlZone(String deviceId, int zoneId, ZoneControlRequest request);
  Future<void> acknowledgeAlarm(String deviceId, String alarmCode);
  Future<List<ScadaHistory>> getHistory(String deviceId, DateRange range);
}
```

---

다음 편은 IoT 모니터링 대시보드 제어 화면 구현이다.
