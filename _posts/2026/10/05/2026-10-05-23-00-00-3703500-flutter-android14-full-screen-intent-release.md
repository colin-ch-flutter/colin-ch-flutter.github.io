---
layout: post
title: "Flutter Android 14 full-screen intent 출시 오류 - USE_FULL_SCREEN_INTENT와 Play 선언 조건"
description: "Flutter Android 14에서 전체 화면 알림이 동작하지 않거나 Play 출시가 거절될 때 USE_FULL_SCREEN_INTENT의 사용 조건과 대체 처리 방법을 정리한다."
date: 2026-10-05
tags: [Flutter, Android, 배포, 알림, CI/CD]
comments: true
share: true
---

![Flutter Android 14 full-screen intent 출시 점검](https://images.unsplash.com/photo-1512428559087-560fa5ceab42?auto=format&fit=crop&w=1600&q=80)

Flutter 앱에서 잠금 화면을 깨우는 전체 화면 알림이 Android 14 release에서 사라진다면 `USE_FULL_SCREEN_INTENT` 한 줄을 추가하는 것보다 먼저 앱의 핵심 기능을 판단해야 한다. 알람 시계나 전화·영상 통화 앱이면 Play Console 선언과 권한 상태를 맞추는 선택이 유리하다. 단순 배송 알림, IoT 경보, 일반 채팅처럼 긴급해 보여도 앱의 핵심 기능이 아닌 경우에는 이 권한을 빼고 일반 고우선 알림으로 낮추는 편이 출시 거절과 사용자 방해를 줄인다.

## Android 14에서 달라진 조건

Android 14(API 34) 이상을 대상으로 하면 `USE_FULL_SCREEN_INTENT`는 일반 manifest 권한처럼 항상 허용되지 않고 특수 앱 액세스 권한으로 취급된다. Google Play의 현재 안내 기준으로 자동 허용 대상은 앱의 핵심 기능이 알람 설정 또는 전화·영상 통화인 경우다. 그 외 앱은 사용자가 설정에서 허용해야 하며, Play Console의 선언 대상이 될 수 있다.

| 앱 기능 | 출시 판단 | 권장 처리 |
|---|---|---|
| 알람 시계 | 자동 허용 심사 대상 | 권한 선언과 Play Console App content 제출 |
| 전화·영상 통화 | 자동 허용 심사 대상 | 수신 화면과 통화 기능을 실제 핵심 흐름으로 유지 |
| IoT 경보·배송·마케팅 | 자동 허용 대상 아님 | 전체 화면 권한 제거, 일반 알림·고우선 채널 사용 |

핵심 기능이 아닌데 manifest에 권한만 남겨두면 로컬 debug에서는 화면이 뜨더라도 Play 업로드 단계에서 `You must let us know whether your app uses any full-screen intent permissions` 같은 오류를 만날 수 있다. 반대로 권한을 무조건 삭제하면 알람 앱의 잠금 화면 수신 흐름이 깨진다.

## Flutter Android 설정을 나누는 기준

알람이나 통화 앱에서만 아래 manifest 권한을 넣는다. 일반 알림 앱이라면 이 블록 자체를 추가하지 않는다.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
```

`flutter_local_notifications` 같은 플러그인에서 전체 화면 알림 옵션을 켜더라도 Android 시스템의 권한 허용 여부가 별도로 남는다. Android 14 기기에서 권한이 꺼진 경우를 정상 상태로 보고, 앱을 열어 일반 알림으로 안내하거나 사용자가 설정에서 선택하도록 만들어야 한다. 앱 시작 때 전체 화면을 강제로 띄우도록 우회하면 알림 차단이나 Play 정책 검토에서 더 불리하다.

출시 전에는 merged manifest와 실제 AAB를 함께 확인한다. 다른 플러그인이 권한을 끌고 들어오는지 확인하지 않으면 Dart 코드에서 사용하지 않는데도 Play Console 선언이 생길 수 있다.

```bash
flutter clean
flutter pub get
flutter build appbundle --release

# 최종 merged manifest에 권한이 들어갔는지 확인
rg "USE_FULL_SCREEN_INTENT" build/app/intermediates/merged_manifests/release -g '*.xml'

# 테스트 기기에서 특수 앱 액세스 상태 확인
adb shell appops get com.example.app USE_FULL_SCREEN_INTENT
```

여기서 `com.example.app`은 실제 `applicationId`로 바꾼다. `allow`가 나와도 Play의 자동 허용 자격을 뜻하지는 않는다. 로컬 기기의 사용자 설정과 Play 배포 심사는 서로 다른 단계다.

## Play Console에서 막힐 때의 복구 순서

1. App Bundle의 merged manifest에서 `USE_FULL_SCREEN_INTENT`를 누가 추가했는지 찾는다.
2. 알람 또는 전화·영상 통화가 앱의 핵심 기능인지 제품 설명과 실제 첫 화면 흐름으로 판단한다.
3. 해당하면 Play Console의 `App content`에서 full-screen intent 선언을 제출하고, 알림이 실제 통화·알람 화면으로 연결되는지 확인한다.
4. 해당하지 않으면 manifest와 플러그인 설정에서 권한을 제거한 뒤 새 AAB를 만든다.
5. 권한이 거부된 상태에서도 제목·소리·탭 이동이 남도록 일반 알림 경로를 테스트한다.

확인 날짜는 2026년 10월 5일이다. Google Play 정책과 콘솔 화면은 바뀔 수 있으므로 제출 직전에 현재 선언 항목을 다시 확인해야 한다. 특히 IoT 앱의 긴급 이벤트는 사용자에게 중요해도 Android의 자동 허용 조건인 알람·통화와 같다고 가정하면 안 된다.

## 출시 체크리스트

- [ ] `targetSdk`가 34 이상인지 기록했다.
- [ ] `USE_FULL_SCREEN_INTENT`를 추가한 플러그인과 manifest를 찾았다.
- [ ] 알람·통화 앱만 Play Console 선언을 제출했다.
- [ ] 권한 거부 상태에서 일반 알림이 표시된다.
- [ ] 잠금 화면, 앱 종료, 재부팅 후 알람·수신 전화 흐름을 별도로 확인했다.
- [ ] 새 AAB의 merged manifest와 Play 업로드 결과를 함께 기록했다.

핵심은 전체 화면 알림을 “더 눈에 띄는 알림”으로 사용하지 않는 것이다. 앱의 핵심 기능이 허용 대상이면 선언과 사용자 설정을 맞추고, 그 외에는 권한을 구매하듯 추가하지 말고 일반 알림으로 설계를 낮추는 것이 Flutter 출시 비용과 정책 리스크를 함께 줄이는 방법이다.

참고 문서: [Google Play full-screen intent 요구사항](https://support.google.com/googleplay/android-developer/answer/13392821), [민감 정보에 접근하는 권한과 API 정책](https://support.google.com/googleplay/android-developer/answer/16558241), [Android NotificationManagerCompat](https://developer.android.com/reference/androidx/core/app/NotificationManagerCompat)
