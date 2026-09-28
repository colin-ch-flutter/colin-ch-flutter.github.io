---
layout: post
title: "Flutter Android 14 포그라운드 서비스 크래시 - 서비스 타입과 권한을 출시 전에 맞추는 법"
description: "Flutter 백그라운드 작업이 Android 14 이상 release에서 SecurityException으로 죽는다면 포그라운드 서비스 타입, 선언 권한, 런타임 조건을 어떻게 점검해야 하는지 정리한다."
date: 2026-09-28
tags: [Flutter, Android, 출시, 백그라운드, IoT, BLE]
comments: true
share: true
---

![Flutter Android 포그라운드 서비스 출시 점검](https://images.unsplash.com/photo-1511707171634-5f897ff02aa9?auto=format&fit=crop&w=1600&q=80)

Android 14(API 34) 이상에서만 Flutter 백그라운드 작업이 시작 직후 죽는다면, Dart 코드보다 포그라운드 서비스의 `type`과 권한을 먼저 맞추는 편이 유리하다. 위치 추적·BLE 동기화처럼 앱이 보이지 않을 때도 계속 실행해야 하는 경우가 대상이고, 단순 주기 작업이나 짧은 네트워크 요청에는 포그라운드 서비스를 추가하면 안 된다. Android 공식 문서 기준으로 서비스 타입, 타입별 권한, 실제 사용 목적이 모두 일치해야 한다.

## debug는 되는데 release에서만 죽는 이유

Android 14부터 `targetSdkVersion`이 34 이상인 앱은 `startForeground()` 호출 시 서비스 타입 검사를 받는다. 선언이 없으면 `MissingForegroundServiceTypeException`, 타입은 있지만 권한이 없거나 런타임 전제가 맞지 않으면 `SecurityException`이 발생할 수 있다. Flutter에서는 이 호출을 `flutter_foreground_task`, 백그라운드 위치, BLE 관련 플러그인의 Android 서비스가 대신하므로 Dart 예외처럼 보이지 않는 경우가 많다.

| 실제 작업 | Manifest 서비스 타입 예 | 추가로 확인할 권한·조건 |
|---|---|---|
| GPS 위치 추적 | `location` | `FOREGROUND_SERVICE_LOCATION`과 위치 런타임 권한 |
| 파일 업로드·동기화 | `dataSync` | `FOREGROUND_SERVICE_DATA_SYNC`, 작업 목적이 데이터 전송인지 |
| 운동·건강 측정 | `health` | `FOREGROUND_SERVICE_HEALTH`, 센서 권한과 허용된 사용 사례 |
| 화면 캡처·공유 | `mediaProjection` | `FOREGROUND_SERVICE_MEDIA_PROJECTION`과 사용자 동의 |

타입 이름만 바꿔서 모든 백그라운드 작업을 통과시킬 수는 없다. Android 14 문서는 타입마다 필요한 권한과 런타임 조건을 별도로 둔다.

## 서비스 선언과 권한을 함께 맞춘다

위치 추적 플러그인이 등록하는 서비스 이름을 먼저 확인한 뒤, 앱 Manifest의 권한과 서비스 속성을 같은 타입으로 맞춘다. 아래는 `location` 작업을 가정한 최소 예시다.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest ...>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

    <application ...>
        <service
            android:name="com.example.background.LocationService"
            android:exported="false"
            android:foregroundServiceType="location" />
    </application>
</manifest>
```

핵심은 서비스 클래스 이름을 임의로 추가하는 것이 아니다. 사용하는 플러그인의 문서와 merged manifest에서 실제 서비스가 등록되는지 확인해야 한다. 플러그인이 이미 같은 서비스를 선언한다면 앱 Manifest에서 타입을 덮어쓰는 방식이 필요할 수 있고, 서비스 이름이 다르면 위 예시를 그대로 복사해도 아무 효과가 없다.

위치 권한도 Manifest 선언만으로 끝나지 않는다. 앱이 보이는 상태에서 사용자의 위치 권한을 얻은 뒤 서비스를 시작해야 하며, 백그라운드 위치가 실제 요구사항이면 `ACCESS_BACKGROUND_LOCATION`의 별도 정책과 UX까지 검토해야 한다. 서비스 타입을 `location`으로 선언했다고 백그라운드 위치 권한이 자동으로 생기지는 않는다.

## release 산출물 기준으로 재현한다

로컬 `flutter run`이 성공했다는 사실은 Play 설치본의 서명·target SDK·merged manifest가 같다는 뜻이 아니다. 출시 전에 다음 순서로 확인한다.

```bash
flutter build apk --release
apkanalyzer manifest permissions build/app/outputs/flutter-apk/app-release.apk
apkanalyzer manifest print build/app/outputs/flutter-apk/app-release.apk \
  | rg "foregroundService|FOREGROUND_SERVICE|LocationService"
adb install -r build/app/outputs/flutter-apk/app-release.apk
adb logcat -c
adb logcat AndroidRuntime:E ActivityManager:E *:S
```

`apkanalyzer`가 없다면 Android Studio의 APK Analyzer에서 `AndroidManifest.xml`을 열어도 된다. 로그에 `requires permissions`가 나오면 타입별 권한이 빠진 것이고, `Starting FGS without a type`라면 서비스 선언의 `android:foregroundServiceType`이 최종 APK에 들어가지 않은 것이다. 이 구분을 하지 않고 Dart에서 재시도만 걸면 크래시 루프가 길어진다.

## 출시 전 체크리스트

- [ ] 실제 작업을 `location`, `dataSync`, `health`, `mediaProjection` 중 하나로 설명할 수 있다.
- [ ] `android:foregroundServiceType`이 플러그인의 실제 서비스에 적용됐다.
- [ ] 기본 `FOREGROUND_SERVICE`와 타입별 권한이 함께 merged manifest에 있다.
- [ ] 서비스 시작 전에 필요한 런타임 권한과 사용자 동의를 받는다.
- [ ] debug가 아닌 `flutter build appbundle --release`와 Play 내부 테스트 트랙에서 확인한다.
- [ ] 알림 권한 거부, 앱 강제 종료, 재부팅 후 시작처럼 실제 운영 경로를 별도로 시험한다.

무료로 해결되는 문제라며 특정 백그라운드 플러그인을 바로 구매하거나 교체할 필요는 없다. 현재 플러그인이 Android 14 타입을 지원하지 않는다면 Manifest만 고쳐서는 해결되지 않으므로, 플러그인 업데이트·네이티브 서비스 수정·WorkManager 전환 중 작업 성격에 맞는 선택을 해야 한다. 짧은 동기화라면 포그라운드 서비스 대신 WorkManager가 더 적합할 수 있다.

짧게 정리하면 Android 14 포그라운드 서비스 오류는 `권한 한 줄` 문제가 아니다. 실제 서비스의 타입, 타입별 권한, 런타임 전제, release APK의 merged manifest를 한 묶음으로 확인해야 한다. 특히 로컬 release APK와 Play App Signing 설치본이 다르면 서명이나 배포 경로까지 분리해 기록해야 한다.

참고 문서: [Android 14 포그라운드 서비스 타입 공식 문서](https://developer.android.com/about/versions/14/changes/fgs-types-required), [Android 포그라운드 서비스 개요](https://developer.android.com/develop/background-work/services/fgs), [Flutter Android 출시 문서](https://docs.flutter.dev/deployment/android)
