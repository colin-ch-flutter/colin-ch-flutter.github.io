---
layout: post
title: "Flutter Android 17 백그라운드 오디오 오류 - 화면을 끄면 재생이 조용히 멈출 때"
description: "Flutter 오디오 앱이 Android 17에서 화면을 끄거나 백그라운드로 보내면 재생과 오디오 포커스가 조용히 실패하는 조건, mediaPlayback 포그라운드 서비스와 ADB 재현 방법을 정리한다."
date: 2026-10-17
tags: [Flutter, Android, 성능최적화, 테스트]
comments: true
share: true
---

![Flutter Android 17 백그라운드 오디오와 재생 제어](https://images.unsplash.com/photo-1478737270239-2f02b77fc618?auto=format&fit=crop&w=1600&q=80)

Android 17에서 화면을 끈 뒤에도 음악·팟캐스트를 재생해야 하는 Flutter 앱은 `AudioPlayer` 호출만 유지해서는 부족하다. 사용자 탭으로 시작한 재생을 계속할 앱이라면 `mediaPlayback` 포그라운드 서비스를 앱이 보이는 동안 시작해야 한다. 화면 안 효과음만 재생한다면 이 대응은 필요 없다.

## 증상은 예외가 아니라 무음 실패다

Android 17의 백그라운드 오디오 강화는 Android 17에서 실행되는 앱 전체에 적용된다. 보이는 Activity가 없고 `SHORT_SERVICE`가 아닌 포그라운드 서비스도 없으면 재생·볼륨 변경 API가 조용히 무시될 수 있고, 오디오 포커스는 `AUDIOFOCUS_REQUEST_FAILED`가 된다. `try/catch`만 추가해도 잡히지 않는 실패다.

| 사용 상황 | 필요한 조건 | 판단 |
|---|---|---|
| 화면 안 효과음·미리듣기 | 보이는 Activity | 별도 백그라운드 서비스 불필요 |
| 잠금 화면 음악·팟캐스트 | `mediaPlayback` FGS | 사용자 재생 시 foreground에서 시작 |
| 부팅 직후 자동 재생 | 사용자 표시·명시적 동작 없음 | Android 17에서 억제될 수 있음 |
| `targetSdk 37` 백그라운드 재생 | WIU 조건을 얻은 FGS | 백그라운드에서 뒤늦게 시작하지 않음 |

## Flutter 플러그인보다 서비스 시작 시점을 먼저 본다

`audio_service` 같은 플러그인을 쓴다면 해당 버전이 `mediaPlayback` 서비스와 미디어 알림을 지원하는지 확인한다. 직접 Android 서비스를 연결했다면 서비스 타입과 권한을 매니페스트에 맞춘다.

```xml
<!-- 실제 서비스 클래스 이름은 사용하는 플러그인 문서를 따른다. -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<service
    android:name="서비스가_제공하는_클래스"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false" />
```

앱 시작 시 무조건 서비스를 띄우지 말고, 재생 버튼을 누른 순간 보이는 Activity에서 재생과 서비스를 함께 시작한다. 버퍼링이나 `AUDIOFOCUS_LOSS_TRANSIENT`에서는 서비스를 유지할 수 있지만, 재생이 끝나면 미디어 세션과 서비스도 종료한다.

## Android 17 에뮬레이터에서 강제로 재현한다

공식 하드닝 토글로 실패 조건을 재현할 수 있다. `throw`는 조용한 실패를 드러낸다.

```bash
adb shell cmd audio set-enable-hardening throw
flutter run --release
# 앱에서 재생 시작 → 홈 버튼 → 화면 잠금 또는 최근 앱에서 제거
adb shell dumpsys audio | rg "AudioHardening|AUDIOFOCUS"
adb logcat -d | rg "AudioHardening|AUDIOFOCUS_REQUEST_FAILED"
```

`level: partial`은 FGS가 없는 경우고, `level: full`은 FGS는 있지만 WIU 조건이 없는 경우다.

```bash
adb shell cmd audio set-enable-hardening disable
```

Android 17에서만 문제가 나타나도 이상하지 않다. Flutter 스트림보다 서비스 수명과 사용자 시작 시점을 먼저 비교한다.

## 출시 전 체크리스트

- [ ] 효과음과 잠금 화면 재생을 분리했다.
- [ ] `mediaPlayback` 서비스가 사용자 동작으로 시작된다.
- [ ] Android 17 하드닝 테스트와 중지·재개를 확인했다.

Android 공식 기준은 2026-10-17에 확인했다. 백그라운드 오디오가 없는 앱에는 이 변경이 적용되지 않으며, 플러그인을 추가했다고 자동 호환되는 것도 아니다. [Android 17 백그라운드 오디오 강화 문서](https://developer.android.com/about/versions/17/changes/bg-audio)와 플러그인 서비스 구성을 함께 확인해야 한다.
