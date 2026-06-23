---
layout: post
title: "BluFi 프로토콜이란? - ESP32 기기에 WiFi 정보 전달하기"
description: " "
date: 2026-05-07
tags: [Flutter, BLE, BluFi, ESP32, WiFi프로비저닝]
comments: true
share: true
---

# BluFi 프로토콜이란? - ESP32 기기에 WiFi 정보 전달하기

![WiFi 라우터](https://images.unsplash.com/photo-1558618047-3c8c76ca7d13?w=800&q=80)

## WiFi 프로비저닝이 왜 필요한가

새 IoT 기기를 처음 설치할 때 기기는 WiFi 정보를 모른다. 기기가 공장에서 출하될 때 사용자 집의 WiFi 비밀번호를 미리 알 수 없기 때문이다.

WiFi 프로비저닝은 "스마트폰이 기기한테 WiFi 정보를 알려주는 과정"이다. 방법은 여러 가지가 있다:

1. **AP 모드**: 기기가 자체 WiFi 핫스팟을 만들고, 스마트폰이 거기 접속해서 정보 전달
2. **BLE**: 블루투스로 WiFi 정보 전달 ← 스마트홈 앱이 쓰는 방식
3. **QR 코드**: QR에 정보를 담아서 카메라로 스캔 (이건 별도 방식)
4. **SmartConfig**: WiFi 패킷으로 정보 전달 (ESP-Touch)

BLE 방식이 가장 안정적이고 사용자 경험도 좋아서 선택했다.

## BluFi 프로토콜 소개

BluFi는 Espressif Systems에서 만든 프로토콜이다. 스마트 IoT 기기에는 ESP32 칩이 들어있고, ESP32 펌웨어에 BluFi 스택이 구현되어 있다.

앱에서는 이 프로토콜에 맞게 BLE 패킷을 만들어서 보내야 한다. Espressif에서 Android/iOS SDK를 제공하지만, Flutter용은 없어서 직접 구현해야 했다.

## 통신 흐름

```
스마트폰 앱                   ESP32 IoT 기기
    |                              |
    |--- BLE 연결 요청 ----------->|
    |<-- 연결 수락 ----------------|
    |                              |
    |--- BluFi 네고시에이션 ------->|
    |<-- 네고시에이션 응답 ---------|
    |                              |
    |--- WiFi SSID 전송 ---------->|
    |--- WiFi Password 전송 ------>|
    |                              |
    |<-- WiFi 연결 시도 결과 ------|
    |                              |
    |--- 서버 등록 API 호출 ------->| (이제 인터넷 됨)
```

## 프로젝트 구조

BLE 관련 코드는 `lib/data/ble/` 에 모여있다:

```
lib/data/ble/
├── blufi_ble_client.dart      # BLE 연결/통신 담당
├── blufi_codec.dart           # 패킷 인코딩/디코딩
├── blufi_constants.dart       # UUID, 상수 정의
└── blufi_packet_assembler.dart # 수신 패킷 조립
```

## BluFi 서비스 UUID

BluFi 프로토콜은 고정된 UUID를 사용한다:

```dart
// blufi_constants.dart
class BlufiConstants {
  // BluFi 서비스 UUID
  static final serviceUuid = Guid('0000ffff-0000-1000-8000-00805f9b34fb');
  
  // 앱 → 기기 (Write)
  static final writeCharUuid = Guid('0000ff01-0000-1000-8000-00805f9b34fb');
  
  // 기기 → 앱 (Notify)
  static final notifyCharUuid = Guid('0000ff02-0000-1000-8000-00805f9b34fb');
}
```

## 패킷 기본 구조

BluFi 패킷 구조:

```
| Type (1B) | Frame Control (1B) | Sequence (1B) | Length (1B) | Data | Checksum (2B) |
```

- **Type**: 패킷 종류 (Ctrl, Data 등)
- **Frame Control**: 단편화 여부, 암호화 여부 등
- **Sequence**: 패킷 순서 번호 (0~255 후 rollover)
- **Length**: Data 길이
- **Data**: 실제 페이로드
- **Checksum**: 오류 검증 (CRC)

## 주요 패킷 종류

```dart
// blufi_constants.dart
class BlufiPacketType {
  // Type = 0 (Control Frame)
  static const int ctrlAck = 0x00;
  static const int ctrlNegotiate = 0x04;
  
  // Type = 1 (Data Frame)
  static const int dataSSID = 0x02;
  static const int dataPassword = 0x03;
  static const int dataConnectAP = 0x06;
  static const int dataOpMode = 0x00;
}
```

WiFi SSID를 보낼 때는 `dataSSID` 타입 패킷을, 비밀번호를 보낼 때는 `dataPassword` 타입 패킷을 만들어서 보낸다.

## 실제 구현의 어려움

솔직히 이 부분을 구현할 때 가장 힘들었다. Espressif 공식 문서가 영어로 되어있는데 설명이 충분하지 않은 부분이 있었고, Flutter로 구현된 레퍼런스가 거의 없었다.

결국 Espressif의 공식 Android SDK 소스 코드를 직접 분석해서 로직을 Dart로 옮겼다. 특히 네고시에이션 과정에서 DH 키 교환을 처리하는 부분이 복잡했다.

---

다음 편은 BluFi 패킷 인코딩/디코딩 코드를 직접 살펴본다.
