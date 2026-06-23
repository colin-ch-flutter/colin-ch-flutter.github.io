---
layout: post
title: "permission_handler로 권한 요청 UX 개선하기"
description: " "
date: 2026-06-04
tags: [Flutter, permission_handler, 권한, UX, Android, iOS]
comments: true
share: true
---

# permission_handler로 권한 요청 UX 개선하기

![앱 권한](https://images.unsplash.com/photo-1589254065878-42c9da997008?w=800&q=80)

## IoT 앱에 필요한 권한들

스마트홈 앱이 필요한 권한을 정리하면:

- **블루투스**: 기기 등록 시 BLE 스캔/연결
- **카메라**: QR 코드 스캔
- **위치**: Android 11 이하에서 BLE 스캔에 필요, Android 12+는 BLUETOOTH_SCAN으로 대체
- **알림**: 푸시 알림 수신 (Android 13+)
- **정확한 알람**: 예약 알림 (Android 12+)

각 권한을 언제 요청하는지도 중요하다. 앱 실행과 동시에 모든 권한을 요청하면 사용자가 거부하는 경우가 많다. 권한이 실제로 필요한 순간에 요청하는 게 좋다.

## permission_handler 기본 사용법

```dart
import 'package:permission_handler/permission_handler.dart';

// 현재 상태 확인
final status = await Permission.bluetooth.status;

// 권한 요청
final result = await Permission.bluetooth.request();

switch (result) {
  case PermissionStatus.granted:
    // 허용됨
    break;
  case PermissionStatus.denied:
    // 거부됨 (다시 요청 가능)
    break;
  case PermissionStatus.permanentlyDenied:
    // 영구 거부 (설정에서만 허용 가능)
    openAppSettings();
    break;
  default:
    break;
}
```

## 권한 요청 전 이유 설명

권한을 바로 요청하기 전에 왜 필요한지 설명하는 다이얼로그를 보여주면 허용률이 올라간다:

```dart
Future<bool> requestBluetoothPermission() async {
  // 이미 허용된 경우
  final status = await Permission.bluetoothScan.status;
  if (status.isGranted) return true;

  // 영구 거부된 경우
  if (status.isPermanentlyDenied) {
    await _showPermanentlyDeniedDialog('블루투스');
    return false;
  }

  // 요청 전 이유 설명
  final shouldRequest = await Get.dialog<bool>(
    AlertDialog(
      title: const Text('블루투스 권한 필요'),
      content: const Text(
        'IoT 기기 기기를 등록하려면 블루투스 연결이 필요합니다.\n'
        '다음 화면에서 블루투스 사용을 허용해주세요.',
      ),
      actions: [
        TextButton(onPressed: () => Get.back(result: false), child: const Text('취소')),
        ElevatedButton(onPressed: () => Get.back(result: true), child: const Text('계속')),
      ],
    ),
  );

  if (shouldRequest != true) return false;

  // 실제 권한 요청
  final result = await _requestBlePermissions();
  return result;
}
```

## Android 버전별 BLE 권한 처리

Android 12부터 BLE 권한이 바뀌었다. 버전에 따라 다르게 요청해야 한다:

```dart
Future<bool> _requestBlePermissions() async {
  if (Platform.isIOS) {
    final result = await Permission.bluetooth.request();
    return result.isGranted;
  }

  // Android
  if (await _isAndroid12OrAbove()) {
    // Android 12+: 위치 권한 불필요
    final results = await [
      Permission.bluetoothScan,
      Permission.bluetoothConnect,
    ].request();

    return results.values.every((s) => s.isGranted);
  } else {
    // Android 11 이하: 위치 권한 필요
    final results = await [
      Permission.bluetooth,
      Permission.location,
    ].request();

    return results.values.every((s) => s.isGranted);
  }
}

Future<bool> _isAndroid12OrAbove() async {
  if (!Platform.isAndroid) return false;
  final androidInfo = await DeviceInfoPlugin().androidInfo;
  return androidInfo.version.sdkInt >= 31;
}
```

## 영구 거부 시 설정 화면으로 안내

```dart
Future<void> _showPermanentlyDeniedDialog(String permissionName) async {
  await Get.dialog(
    AlertDialog(
      title: Text('$permissionName 권한 필요'),
      content: Text(
        '$permissionName 권한이 거부되어 있습니다.\n'
        '설정 > 앱 > 스마트홈 > 권한에서\n'
        '$permissionName을 허용해주세요.',
      ),
      actions: [
        TextButton(onPressed: Get.back, child: const Text('닫기')),
        ElevatedButton(
          onPressed: () {
            Get.back();
            openAppSettings(); // 설정 앱으로 이동
          },
          child: const Text('설정 열기'),
        ),
      ],
    ),
  );
}
```

## iOS Info.plist 권한 설명 문구

iOS는 권한 설명 문구를 `Info.plist`에 미리 작성해야 한다. 앱 심사에서 이 문구가 적절한지 검토한다:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>IoT 기기 기기 등록 및 WiFi 설정을 위해 블루투스 접근이 필요합니다</string>

<key>NSCameraUsageDescription</key>
<string>QR 코드로 기기를 등록하기 위해 카메라 접근이 필요합니다</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>주변 기기 검색을 위해 위치 접근이 필요합니다</string>
```

---

다음 편은 flutter_secure_storage로 민감 정보를 안전하게 저장하는 방법이다.
