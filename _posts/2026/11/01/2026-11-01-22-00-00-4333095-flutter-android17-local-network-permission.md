---
layout: post
title: "Flutter Android 17 로컬 네트워크 오류 - ACCESS_LOCAL_NETWORK 권한과 Socket timeout"
description: "Flutter 앱이 Android 17 출시 뒤 로컬 기기에 연결하지 못하고 Socket timeout을 낼 때, ACCESS_LOCAL_NETWORK 선언·런타임 요청·거부 처리를 target SDK별로 점검한다."
date: 2026-11-01
tags: [Flutter, Android, IoT, 네트워크, 배포]
comments: true
share: true
---

![Flutter Android 17 로컬 네트워크 권한과 기기 연결 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

Android 17(API 37)을 target으로 올린 Flutter 앱이 같은 Wi-Fi의 ESP32, NAS, 프린터에 연결되지 않는다면 `INTERNET` 권한만 확인해서는 부족하다. 앱의 핵심 기능이 LAN 기기와 직접 통신하는 경우 `ACCESS_LOCAL_NETWORK`를 선언하고, Dart socket을 열기 전에 런타임 권한을 요청해야 한다. 반대로 인터넷 API만 호출하는 앱이나 target SDK 36 이하 앱에는 이 권한을 추가할 필요가 없다.

## 오류가 timeout으로 보이는 이유

Android 17부터 target SDK 37 이상 앱의 로컬 네트워크 접근은 기본 차단된다. Flutter 공식 문서가 설명하듯 `dart:io`의 `Socket`은 Android 권한 팝업을 직접 띄우지 못한다. 그래서 권한이 없을 때 명확한 “permission denied” 대신 연결이 오래 기다리다 `SocketException` 또는 timeout으로 끝날 수 있다.

| 앱 조건 | 필요한 처리 | 판단 |
|---|---|---|
| target 36 이하 | 기존 `INTERNET` 사용 | `ACCESS_LOCAL_NETWORK`를 추가하지 않는다 |
| target 37 이상 + LAN 직접 연결 | manifest 선언 + 런타임 요청 | 권한 승인 뒤 socket을 연다 |
| Cast 등 시스템 picker로 연결 | picker 우선 검토 | 광범위한 권한이 불필요할 수 있다 |

## manifest와 Flutter 요청 순서

권한을 요청하기 전에 Android manifest에 선언해야 한다. `permission_handler`는 `ACCESS_LOCAL_NETWORK` 지원 버전인지 확인해야 하며, 확인한 changelog 기준 지원은 13.0.2부터다.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />

    <application
        android:label="my_app"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        ...
    </application>
</manifest>
```

권한 승인을 확인한 뒤에만 `Socket.connect`를 호출하도록 연결 진입점을 묶는다.

```dart
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:permission_handler/permission_handler.dart';

Future<Socket?> connectToDevice(String host, int port) async {
  if (Platform.isAndroid) {
    final status = await Permission.accessLocalNetwork.request();
    if (status.isPermanentlyDenied) {
      debugPrint('LAN 권한이 영구 거부됐다. 설정 화면을 안내한다.');
      await openAppSettings();
      return null;
    }
    if (!status.isGranted) return null;
  }

  try {
    return await Socket.connect(
      host,
      port,
      timeout: const Duration(seconds: 5),
    );
  } on SocketException catch (error) {
    debugPrint('LAN 연결 실패: $error');
    return null;
  }
}
```

이 코드는 권한을 매번 강제로 띄우는 코드가 아니다. 앱 시작 때 무조건 요청하기보다 사용자가 “기기 추가”를 눌렀을 때 설명을 보여주고 요청하는 편이 낫다. 거부됐으면 재시도 안내를, 영구 거부됐으면 앱 설정 이동을 제공해야 한다. 권한이 승인돼도 기기의 IP, 포트, 방화벽, 같은 네트워크 여부가 틀리면 연결은 실패한다.

## 출시 전에 확인할 조건

Android 16(API 36)에서 동작하는 앱이 target 37로 바뀌는 순간을 별도로 시험해야 한다. 최소한 아래 순서로 로그를 남긴다.

- `flutter build appbundle --release`가 실제 target SDK 37로 생성됐는지 확인한다.
- 새 설치 상태에서 권한 승인·거부·설정에서 철회한 세 경우를 각각 실행한다.
- `192.168.x.x` 같은 LAN 주소와 공인 HTTPS 주소를 구분해 시험한다.
- 권한을 승인했는데도 timeout이면 AP 격리, 포트 미개방, 잘못된 IPv6 주소를 확인한다.
- Play Console 제출 전 `ACCESS_LOCAL_NETWORK`가 앱의 핵심 기능에 필요한지 설명할 수 있게 만든다.

Google Play는 2026년 10월 발표한 정책에서 target 37 이상 앱의 이 권한을 핵심 기능과 연결하고, 시스템 picker로 대체할 수 없는 이유를 요구하는 방향을 제시했다. 단순히 “스캔 기능이 편해서” 추가하는 권한으로 보면 출시 심사와 정책 대응에서 불리하다.

정리하면 Android 17의 Flutter LAN 오류는 `http` 패키지 교체 문제가 아니다. target SDK를 확인하고, 직접 통신이면 manifest 선언과 `Permission.accessLocalNetwork` 요청을 연결 전에 수행한다. 구매할 네이티브 네트워크 SDK 없이도 이 흐름으로 해결할 수 있지만, picker가 가능한 기능까지 광범위한 권한을 요청하지 않는 선택이 더 안전하다.

공식 기준은 2026년 11월 1일 확인했다. [Flutter Android 로컬 네트워크 권한 문서](https://docs.flutter.dev/platform-integration/android/local-network-permission), [Android 로컬 네트워크 권한 문서](https://developer.android.com/privacy-and-security/local-network-permission), [Google Play 권한 정책](https://support.google.com/googleplay/android-developer/answer/16935362)을 함께 확인해야 한다.
