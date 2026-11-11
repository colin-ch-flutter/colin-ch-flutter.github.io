---
layout: post
title: "Flutter Android 17 백그라운드 오디오 무음 오류 - audio_service 출시 조건"
description: "Flutter 앱이 Android 17에서 화면을 끄거나 백그라운드로 보낸 뒤 오디오만 무음이 될 때, mediaPlayback 포그라운드 서비스와 audio_service 설정·재현 명령을 점검하는 방법을 정리한다."
date: 2026-11-11
tags: [Flutter, Android, 오디오, 배포, 출시]
comments: true
share: true
---

![Flutter Android 17 백그라운드 오디오 재생과 포그라운드 서비스](https://images.unsplash.com/photo-1505740420928-5e560c06d30e?auto=format&fit=crop&w=1600&q=80)

이 그림에서는 화면이 꺼져도 재생을 계속하려면 Flutter UI가 아니라 Android 미디어 서비스가 재생 생명주기를 맡아야 한다는 점을 보면 된다.

Android 17(API 37)을 대상으로 하는 Flutter 앱에서 화면을 나간 뒤 소리가 끊기거나 무음이 되면 `just_audio`의 URL이나 코덱부터 바꿀 문제가 아닐 수 있다. 백그라운드 재생이 제품 기능이면 `audio_service` 같은 미디어 세션 구조와 `mediaPlayback` 포그라운드 서비스를 선택하는 편이 맞다. 화면이 보일 때만 효과음·알림음을 재생하는 앱에는 이 구성이 필요하지 않다.

## Android 17에서 달라진 실패 모양

Android 공식 문서 기준으로 Android 17은 백그라운드 오디오 재생, 오디오 포커스, 볼륨 API를 더 엄격하게 제한한다. 특히 target SDK 37 앱이 백그라운드에서 오디오를 만질 때는 포그라운드 서비스가 필요하고, 서비스도 사용 중인 앱에서 시작되어 WIU(while-in-use) 조건을 갖춰야 한다.

그래서 “재생 상태는 playing인데 소리만 안 난다”거나 “예외 없이 백그라운드에서 멈춘다”는 증상이 가능하다. 아래처럼 조건을 나누면 원인을 빠르게 좁힐 수 있다.

| 상황 | 판단 | 대응 |
| --- | --- | --- |
| 앱 화면 안에서만 재생 | Android 17 hardening 영향이 작음 | 일반 `just_audio` 유지 |
| 화면을 끄면 계속 재생 | 백그라운드 미디어 기능 | `audio_service` 또는 `just_audio_background` 검토 |
| 부팅 직후 자동 재생 | 사용자 동작 없는 백그라운드 시작 | 자동 재생 정책과 서비스 시작 시점을 재설계 |
| 서비스는 있지만 무음 | FGS 타입·WIU·시작 시점 점검 | manifest와 logcat을 함께 확인 |

## Flutter 프로젝트에서 확인할 설정

`audio_service`를 사용하는 경우 Android 매니페스트에 미디어 재생용 권한과 서비스를 등록한다. 패키지 버전에 따라 예제의 클래스명이나 Activity 연결 방식이 달라질 수 있으므로, 아래 블록은 현재 패키지 문서의 핵심 항목만 옮긴 것이다.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<application ...>
    <service
        android:name="com.ryanheise.audioservice.AudioService"
        android:foregroundServiceType="mediaPlayback"
        android:exported="true">
        <intent-filter>
            <action android:name="android.media.browse.MediaBrowserService" />
        </intent-filter>
    </service>
</application>
```

재생을 시작하는 버튼도 서비스가 백그라운드에서 처음 뜨도록 두지 말고, 사용자가 앱을 보고 누른 시점에 연결해야 한다. `BaseAudioHandler`의 재생 상태와 실제 `AudioPlayer` 상태가 따로 놀면 서비스는 foreground인데 소리는 멈추는 식의 오진이 생기므로, `playing`, `processingState`, 오류 이벤트를 같은 로그에 남기는 편이 좋다.

## Android 17 에뮬레이터에서 재현하기

Android 공식 문서가 안내하는 hardening 토글로 기존 기기에서도 실패 조건을 확인할 수 있다. 아래 명령은 앱 전체의 오디오 제한을 강제하므로 테스트 기기에서만 실행한다.

```bash
adb shell cmd audio set-enable-hardening throw
adb logcat | rg 'AudioHardening|AndroidRuntime|AudioTrack'
```

`throw`는 무음 실패를 예외나 오류 코드로 드러내는 모드다. `level: partial`이면 포그라운드 서비스가 없고, `level: full`이면 서비스는 있지만 WIU 능력이 없는 경우를 의심할 수 있다. 수정 뒤에는 `enable` 모드에서 화면 잠금, 홈 이동, 알림 재생, 블루투스 연결을 각각 확인한다.

## 출시 전 체크리스트

- [ ] target SDK가 37일 때만 실패하는지 Android 16과 비교했다.
- [ ] 사용자가 화면 안에서 재생을 시작한 뒤 서비스가 foreground가 되는지 확인했다.
- [ ] `FOREGROUND_SERVICE_MEDIA_PLAYBACK` 권한과 `mediaPlayback` 타입을 함께 넣었다.
- [ ] `AudioHardening` 로그와 실제 재생 상태를 같은 테스트 케이스로 저장했다.
- [ ] 음악 앱이 아니라 단순 효과음 앱이라면 불필요한 상시 서비스를 추가하지 않았다.

핵심은 `audio_service`를 추가하는 것 자체가 아니라, 백그라운드 재생을 사용자가 시작한 미디어 작업으로 연결하는 데 있다. Android 17 대상 앱에서 무음이 발생하면 코덱 교체보다 서비스 시작 시점, WIU 조건, `mediaPlayback` 선언을 확인하는 순서가 재현과 복구 비용을 줄인다.

기준은 2026년 11월 11일 확인한 [Android 17 백그라운드 오디오 hardening 문서](https://developer.android.com/about/versions/17/changes/bg-audio), [Android 17 동작 변경 문서](https://developer.android.com/about/versions/17/behavior-changes-17), [`audio_service` 패키지 문서](https://pub.dev/packages/audio_service)다. Flutter 플러그인의 매니페스트 요구사항은 버전에 따라 바뀔 수 있으므로 실제 의존성의 README와 함께 대조해야 한다.
