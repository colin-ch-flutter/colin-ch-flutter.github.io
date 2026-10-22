---
layout: post
title: "Flutter Android 17 로컬 네트워크 오류 - Dart SocketException 출시 조건"
description: "Flutter 앱이 Android 17에서 192.168.x.x 장비에 연결되지 않을 때 ACCESS_LOCAL_NETWORK 선언과 permission_handler 요청 순서, 테스트 조건을 정리한다."
date: 2026-10-23
tags: [Flutter, Android, IoT, 스마트홈, 네트워크]
comments: true
share: true
---

![Flutter Android 17 로컬 네트워크 권한과 공유기 연결 점검](https://images.unsplash.com/photo-1606904825846-647eb07f5be2?auto=format&fit=crop&w=1600&q=80)

Android 17(API 37) 대상 Flutter 앱에서 공유기, 프린터, NAS, 스마트홈 장비의 사설 IP에 연결해야 한다면 `Socket.connect` 전에 로컬 네트워크 권한을 처리해야 한다. 권한이 필요한 앱은 `ACCESS_LOCAL_NETWORK`을 선언하고 런타임 요청까지 해야 한다. 반대로 AWS IoT 같은 외부 서버만 연결하거나 Android 16 이하만 대상으로 하는 앱에는 이 변경을 억지로 추가할 필요가 없다.

## 연결 코드가 갑자기 `SocketException`이 되는 조건

Flutter의 `dart:io` `Socket`은 Android 권한 대화상자를 직접 띄우지 못한다. Android 17에서 로컬 IP 접근이 기본 차단된 상태면 예외가 발생한 뒤에야 실패한다. `try/catch`를 추가하는 것만으로는 권한을 얻을 수 없다.

| 연결 대상 | Android 17 로컬 네트워크 권한 | 판단 |
|---|---:|---|
| `192.168.0.20:1883` MQTT 브로커 | 필요 | 앱에서 권한 요청 후 연결 |
| `192.168.0.50:80` NAS HTTP API | 필요 | 스캔·연결 전에 권한 확인 |
| `api.example.com:443` HTTPS API | 불필요 | 일반 인터넷 권한 영역 |
| BLE GATT 연결 | 별도 권한 | 이 글의 LAN 권한과 분리 |

공식 Flutter 문서 기준으로 이 제한은 Android 17을 대상으로 하는 앱에서 적용된다. Android 16에서도 테스트용 opt-in이 가능하므로 출시 전에 두 버전의 결과를 나눠 확인해야 한다.

## Manifest와 런타임 요청을 같이 넣는다

Manifest 선언만 넣으면 권한 대화상자가 자동으로 뜬다고 생각하기 쉽다. 하지만 Dart socket은 시스템 UI를 호출하지 않으므로, 연결 함수가 시작되기 전에 `permission_handler`로 직접 요청해야 한다.

`android/app/src/main/AndroidManifest.xml`의 `<manifest>` 바로 아래에 권한을 선언한다.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />

    <application
        android:label="my_app"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        <!-- ... -->
    </application>
</manifest>
```

다음 코드는 권한을 확인한 뒤에만 로컬 장비 socket을 여는 최소 경계다. 연결 실패와 권한 거부를 같은 오류로 뭉개지 않는 것이 핵심이다.

```dart
import 'dart:io';

import 'package:permission_handler/permission_handler.dart';

Future<Socket> connectToLocalDevice({
  required String host,
  required int port,
}) async {
  if (Platform.isAndroid) {
    final status = await Permission.accessLocalNetwork.request();

    if (!status.isGranted) {
      if (status.isPermanentlyDenied) {
        await openAppSettings();
      }
      throw const LocalNetworkPermissionDenied();
    }
  }

  return Socket.connect(host, port, timeout: const Duration(seconds: 3));
}

class LocalNetworkPermissionDenied implements Exception {
  const LocalNetworkPermissionDenied();
}
```

`Permission.accessLocalNetwork`이 현재 프로젝트의 `permission_handler` 버전에서 인식되지 않으면 패키지 버전만 올리고 끝내지 말고, Android Gradle Plugin·compileSdk·Flutter SDK 조합을 함께 확인한다. 권한 API를 지원하지 않는 버전에서 임의의 문자열로 우회하면 실제 시스템 권한 상태와 Dart 코드가 어긋난다.

## 스캔 전에 요청해야 하는 이유

스마트홈 앱은 흔히 앱 시작 때 MQTT 연결, mDNS 검색, 장비 ping을 동시에 시작한다. 권한 요청을 화면 버튼 안쪽에만 두면 첫 진입 시 이미 socket이 실패한다. 아래처럼 경계를 나누면 재현과 복구가 쉽다.

```text
사용자 탭
  → 로컬 네트워크 권한 확인/요청
  → 권한 승인
  → mDNS·TCP·LAN MQTT 초기화
  → 장비 검색
```

권한을 거부한 사용자는 “장비가 꺼졌다”고 오해할 수 있으므로, 연결 실패 화면에 네트워크 권한 설정으로 이동하는 버튼과 대상 주소를 함께 표시한다. 앱을 재개한 뒤에는 권한이 설정에서 바뀌었을 수 있으므로 `onResume` 시 연결 상태를 다시 확인한다.

## 출시 전 체크리스트

- [ ] `targetSdk`가 37 이상인지 확인했다.
- [ ] Manifest에 `ACCESS_LOCAL_NETWORK`을 선언했다.
- [ ] 로컬 socket·mDNS 검색 전에 권한 요청이 실행된다.
- [ ] Android 16 opt-in과 Android 17 기기에서 허용·거부·영구 거부를 각각 확인했다.
- [ ] 권한 거부를 장비 오프라인과 다른 사용자 메시지로 표시한다.
- [ ] `192.168.x.x` 연결과 외부 HTTPS 연결을 별도 테스트했다.

핵심은 모든 네트워크 권한을 한꺼번에 추가하는 것이 아니다. Android 17에서 사설 IP를 검색하거나 연결하는 기능만 `ACCESS_LOCAL_NETWORK` 경계 안에 두고, 외부 API와 BLE 권한은 별도 상태로 관리해야 한다. 이 조건을 빼먹으면 debug에서 보이던 장비가 release에서만 사라지는 문제가 생길 수 있다.

참고한 공식 문서는 [Flutter의 Android 로컬 네트워크 권한 안내](https://docs.flutter.dev/platform-integration/android/local-network-permission)다. Android 17 권한 동작과 Flutter 문서는 2026-10-23에 확인했다.
