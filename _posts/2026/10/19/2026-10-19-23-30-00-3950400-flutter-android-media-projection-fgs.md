---
layout: post
title: "Flutter Android 14 MediaProjection 오류 - 화면 캡처 Foreground Service 출시 조건"
description: "Flutter 화면 녹화·공유 앱이 Android 14 release에서 SecurityException으로 종료될 때 MediaProjection 동의 순서, Foreground Service 선언, Android 15 부팅 제한을 점검한다."
date: 2026-10-19
tags: [Flutter, Android, 배포·운영, 플랫폼 통합, 테스트]
comments: true
share: true
---

![Flutter Android 화면 캡처와 Foreground Service 출시 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 부분은 Flutter 화면이 아니라 Android 시스템 권한과 백그라운드 실행 경계가 따로 있다는 점이다.

Flutter 앱에서 화면 녹화나 화면 공유가 debug에서는 되다가 Android 14 release에서 `SecurityException`으로 종료된다면, 권한 한 줄보다 실행 순서를 고쳐야 한다. `targetSdk 34` 이상 앱은 `mediaProjection` 전용 Foreground Service 권한과 서비스 타입을 선언하고, 사용자의 화면 캡처 동의를 받은 뒤 서비스를 시작해야 한다. 단순 원격 제어·파일 전송 앱이라면 MediaProjection을 쓰지 않는 선택이 더 안전하다.

## 오류가 발생하는 조건

Android 공식 문서 기준으로 `mediaProjection`은 일반 `dataSync` 서비스와 다른 타입이다. 선언이 빠지면 `MissingForegroundServiceTypeException` 또는 아래와 비슷한 `SecurityException`이 나온다.

| 확인 항목 | 잘못된 상태 | 출시용 기준 |
|---|---|---|
| manifest 권한 | `FOREGROUND_SERVICE`만 있음 | 공통 권한과 `FOREGROUND_SERVICE_MEDIA_PROJECTION`을 함께 선언 |
| 서비스 타입 | `foregroundServiceType` 없음 | `mediaProjection` 지정 |
| 사용자 동의 | 서비스부터 시작 | `createScreenCaptureIntent()` 결과를 받은 뒤 시작 |
| 세션 재사용 | 같은 토큰으로 다시 캡처 | 캡처 세션마다 새 동의 요청 |
| Android 15 부팅 | `BOOT_COMPLETED`에서 캡처 서비스 시작 | 부팅 리시버에서 시작하지 않음 |

`flutter_background` 이슈에도 target SDK 34에서 `FOREGROUND_SERVICE_MEDIA_PROJECTION`이 없어서 서비스 시작이 실패한 로그가 남아 있다. 플러그인이 manifest를 병합하더라도 최종 AAB의 merged manifest와 네이티브 서비스 코드를 확인해야 한다.

## manifest와 시작 순서

화면 캡처 플러그인이 이 선언을 제공하지 않거나 앱이 직접 서비스를 갖고 있다면 `android/app/src/main/AndroidManifest.xml`에 아래처럼 둔다.

```xml
<manifest ...>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROJECTION" />

    <application ...>
        <service
            android:name=".ScreenCaptureService"
            android:exported="false"
            android:foregroundServiceType="mediaProjection" />
    </application>
</manifest>
```

핵심은 Dart에서 `startService`만 호출하지 않는 것이다. Android 쪽에서 사용자 동의 Activity Result를 먼저 받고, 그 결과를 서비스에 전달한 뒤 `startForeground()`를 호출하는 흐름이어야 한다.

```kotlin
// Activity: Flutter MethodChannel 호출을 받은 뒤 실행하는 최소 흐름
private val captureLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == Activity.RESULT_OK && result.data != null) {
        val intent = Intent(this, ScreenCaptureService::class.java)
            .putExtra("projection_result", result.resultCode)
            .putExtra("projection_data", result.data)
        ContextCompat.startForegroundService(this, intent)
    }
}

fun requestCapture() {
    val manager = getSystemService(MediaProjectionManager::class.java)
    captureLauncher.launch(manager.createScreenCaptureIntent())
}
```

서비스가 시작되면 전달받은 데이터로 `getMediaProjection()`을 호출하고, 알림과 `mediaProjection` 타입으로 `startForeground()`를 실행한다. 같은 `MediaProjection` 객체로 `createVirtualDisplay()`를 두 번 호출하거나, 한 번 사용한 동의 Intent를 재사용하면 Android 14 이상에서 다시 `SecurityException`이 날 수 있다.

## Android 15와 종료 처리

Android 15를 대상으로 하는 앱은 `BOOT_COMPLETED` 리시버에서 media projection Foreground Service를 시작할 수 없다. 재부팅 뒤 자동 녹화를 기대하는 설계라면 부팅 시에는 상태만 저장하고, 사용자가 앱 화면에서 다시 캡처 동의를 하도록 바꿔야 한다. 동의 없이 백그라운드에서 몰래 화면을 이어가는 방식은 플랫폼 정책과 사용자 기대 모두에 맞지 않는다.

사용자가 상태 표시줄의 projection 칩을 누르거나 화면을 잠그면 세션이 끝날 수 있다. 네이티브 `MediaProjection.Callback.onStop()`에서 VirtualDisplay와 Surface를 해제하고 Flutter에 `captureStopped` 이벤트를 보내야 한다. 이 콜백을 빠뜨리면 종료된 세션을 다시 만들려다 `IllegalStateException`이 발생한다.

출시 전에는 아래 네 경로를 따로 확인한다.

- [ ] Android 14에서 동의 취소, 허용 후 시작, 앱 전환을 각각 확인했다.
- [ ] Android 15에서 화면 잠금과 상태 표시줄 projection 중지를 확인했다.
- [ ] release AAB의 merged manifest에 두 권한과 `mediaProjection` 타입이 남아 있다.
- [ ] 프로세스 종료 후 같은 토큰을 재사용하지 않고 새 동의를 요청한다.

Android 공식 문서 확인 기준은 2026년 10월 19일이다. [Media projection 권한·동의 조건](https://developer.android.com/media/grow/media-projection), [Foreground Service 타입별 요구사항](https://developer.android.com/develop/background-work/services/fgs/service-types), [Flutter 플러그인에서 발생한 실제 SecurityException 사례](https://github.com/JulianAssmann/flutter_background/issues/92)를 함께 확인하면 manifest 누락과 세션 수명 문제를 구분할 수 있다.

핵심은 `mediaProjection`을 일반 백그라운드 작업처럼 다루지 않는 것이다. Android 14에서는 전용 권한·서비스 타입·사용자 동의 순서를 맞추고, Android 15에서는 부팅 자동 시작을 제거하며, 세션이 끝날 때 네이티브 자원을 해제해야 Flutter release 크래시를 줄일 수 있다.
