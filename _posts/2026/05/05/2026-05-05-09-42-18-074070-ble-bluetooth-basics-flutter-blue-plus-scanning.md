---
layout: post
title: "BLE(블루투스 저에너지) 기초 - flutter_blue_plus로 스캔 시작"
description: " "
date: 2026-05-05
tags: [Flutter, BLE, 블루투스, flutter_blue_plus, IoT]
comments: true
share: true
---

# BLE(블루투스 저에너지) 기초 - flutter_blue_plus로 스캔 시작

![블루투스 IoT](https://images.unsplash.com/photo-1605810230434-7631ac76ec81?w=800&q=80)

## 왜 BLE가 필요한가

새 IoT 기기를 사서 앱에 등록할 때를 생각해보자. IoT 기기는 공장에서 나올 때 WiFi 정보가 없다. WiFi 없이는 클라우드 서버에 연결할 수 없고, 클라우드 서버에 연결 못 하면 앱으로 제어도 안 된다.

이 닭이 먼저냐 달걀이 먼저냐 문제를 해결하는 방법이 BLE를 통한 WiFi 프로비저닝이다. 스마트폰이 BLE로 IoT 기기에 직접 연결해서 "우리 집 WiFi는 이거야, 비밀번호는 이거야" 하고 알려주는 것이다. 그러면 IoT 기기가 WiFi에 연결되고, 이후로는 클라우드를 통해 정상적으로 제어할 수 있다.

## BLE vs 클래식 블루투스

BLE(Bluetooth Low Energy)는 이름대로 저전력 블루투스다.

| 구분 | 클래식 블루투스 | BLE |
|------|----------------|-----|
| 전력 소비 | 높음 | 낮음 |
| 데이터 속도 | 빠름 (24 Mbps) | 느림 (1 Mbps) |
| 용도 | 오디오, 파일 전송 | 센서, IoT |
| 페어링 | 필요 | 불필요 (연결만으로 가능) |

IoT 기기는 배터리로 동작하거나 전력 절약이 중요한 경우가 많아서 BLE를 쓴다.

## flutter_blue_plus 설정

```yaml
dependencies:
  flutter_blue_plus: ^1.35.4
```

### Android 권한 설정

`AndroidManifest.xml`에 권한을 추가해야 한다:

```xml
<!-- Android 12 이상 -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />

<!-- Android 11 이하 (위치 권한이 BLE 스캔에 필요) -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30" />
```

Android 12에서 BLE 권한 모델이 바뀌었다. 12 이전에는 위치 권한이 있어야 BLE 스캔이 됐는데, 12 이후에는 `BLUETOOTH_SCAN`에 `neverForLocation` 플래그를 쓰면 위치 권한 없이도 스캔된다.

### iOS 권한 설정

`Info.plist`에 추가:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>IoT 기기 기기 등록을 위해 블루투스 접근이 필요합니다</string>
```

## BLE 스캔 구현

```dart
// 스캔 시작
FlutterBluePlus.startScan(
  timeout: const Duration(seconds: 10),
  withNames: ['SmartIoT Device'],  // 특정 이름 필터링
);

// 스캔 결과 구독
FlutterBluePlus.scanResults.listen((results) {
  for (ScanResult result in results) {
    print('발견: ${result.device.platformName}');
    print('RSSI: ${result.rssi}dBm');  // 신호 강도
    print('주소: ${result.device.remoteId}');
  }
});

// 스캔 중지
await FlutterBluePlus.stopScan();
```

기기 이름으로 필터링하면 주변의 수많은 BLE 기기 중 IoT 기기만 걸러낼 수 있다.

## 블루투스 상태 확인

스캔 전에 블루투스 활성화 여부를 확인해야 한다:

```dart
// 블루투스 상태 스트림
FlutterBluePlus.adapterState.listen((state) {
  if (state == BluetoothAdapterState.on) {
    startScan();
  } else {
    // 블루투스 켜달라고 안내
    showBluetoothOffDialog();
  }
});
```

## 실제로 겪은 문제들

**Android와 iOS 동작 차이**: Android에서는 스캔 중에 같은 기기가 중복으로 올라오는 경우가 있다. 기기 ID를 키로 Map에 관리하면서 중복을 제거했다.

**iOS에서 앱 재시작 후 이전 연결 처리**: iOS는 앱이 종료되더라도 시스템 레벨에서 BLE 연결 상태를 유지하는 경우가 있다. 앱 재시작 시 이미 연결된 기기가 있는지 확인하는 로직이 필요했다:

```dart
// 이미 연결된 기기 확인
final connected = FlutterBluePlus.connectedDevices;
```

**권한 없을 때 크래시**: 권한 확인 없이 스캔하면 크래시가 난다. 스캔 전에 반드시 `permission_handler`로 권한 상태를 확인해야 한다.

---

다음 편은 BLE 기기 연결 후 GATT 서비스를 통해 데이터를 주고받는 방법이다.
