---
layout: post
title: "Flutter Android 13 알림 권한 오류 - POST_NOTIFICATIONS 출시 빌드 점검"
description: "Flutter 앱에서 Android 13 이상 알림이 보이지 않을 때 POST_NOTIFICATIONS 런타임 권한, FCM 포그라운드 처리, 로컬 알림 채널을 출시 빌드 기준으로 점검하는 방법을 정리한다."
date: 2026-09-22
tags: [Flutter, Android, Firebase, FCM, 푸시알림, 배포]
comments: true
share: true
---

![Flutter Android 알림 권한 점검 흐름](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Android 13(API 33) 이상에서만 알림이 사라지고 Android 12에서는 보인다면 `POST_NOTIFICATIONS` 런타임 권한부터 확인해야 한다. FCM을 쓰는 앱은 `firebase_messaging`으로 권한을 요청하고, 포그라운드에서도 배너를 보여야 한다면 별도의 로컬 알림을 호출해야 한다. `AndroidManifest.xml`에 권한 한 줄만 추가하는 것으로는 출시 빌드 문제가 끝나지 않는다.

## 증상을 두 갈래로 나눈다

FCM 메시지가 도착하지 않는 문제와 도착했지만 화면에 알림이 표시되지 않는 문제는 점검 위치가 다르다. 특히 포그라운드에서는 FCM의 notification payload가 자동으로 시스템 알림을 만들지 않는 것이 정상이다.

| 상황 | 확인 항목 | 흔한 원인 |
|---|---|---|
| Android 13+ 전체에서 안 보임 | 권한 상태와 사용자 거부 여부 | `POST_NOTIFICATIONS` 미요청 |
| 백그라운드에서는 보이고 앱을 열면 안 보임 | `onMessage` 처리 | 포그라운드 표시 코드를 작성하지 않음 |
| 로컬 알림만 안 보임 | 채널 ID와 중요도 | 이미 만들어진 채널의 설정을 코드로 바꾸려 함 |
| Android 12 이하에서만 다름 | target SDK와 채널 생성 시점 | 시스템 자동 권한 대화상자 타이밍 차이 |

Android 공식 문서 기준으로 Android 13부터 앱은 알림 전송 전에 `POST_NOTIFICATIONS` 권한을 받아야 한다. FCM SDK가 manifest 권한을 병합하더라도 사용자의 런타임 허용은 별도다.

## FCM 권한 요청을 앱 준비 단계에 둔다

FCM 토큰을 받기 전에 권한을 확인하면, 출시 후 최초 실행에서 권한 상태와 토큰 등록 시점이 뒤엉키는 문제를 줄일 수 있다. 권한 요청은 앱이 왜 알림을 쓰는지 설명한 화면의 버튼 뒤에 두는 편이 좋다.

아래 코드는 Android와 iOS를 함께 고려하는 FCM 권한 요청 예시다.

```dart
import 'package:firebase_messaging/firebase_messaging.dart';

Future<bool> preparePushPermission() async {
  final settings = await FirebaseMessaging.instance.requestPermission(
    alert: true,
    badge: true,
    sound: true,
    provisional: false,
  );

  final allowed = settings.authorizationStatus ==
          AuthorizationStatus.authorized ||
      settings.authorizationStatus == AuthorizationStatus.provisional;

  if (allowed) {
    final token = await FirebaseMessaging.instance.getToken();
    // 서버에 token을 등록한다.
    return token != null;
  }

  return false;
}
```

Android 13 이상에서는 사용자가 거부했는지와 아직 요청하지 않았는지를 `authorizationStatus`만으로 완전히 구분하기 어렵다는 점도 기록해 둬야 한다. 앱 내부에 “권한 요청을 시도했는가”를 저장하고, 거부 상태라면 설정 화면으로 이동시키는 안내를 제공하는 방식이 안전하다.

## 포그라운드 알림은 직접 표시한다

FCM의 `onMessage`는 앱이 화면에 떠 있을 때 호출되지만 시스템 알림을 자동으로 표시하지 않는다. `flutter_local_notifications`를 함께 사용한다면 Android 13+에서 해당 플러그인의 `requestNotificationsPermission()`도 호출하고, 같은 채널 ID를 초기화와 표시 코드에서 유지한다.

```dart
final local = FlutterLocalNotificationsPlugin();

Future<void> prepareLocalNotification() async {
  const android = AndroidInitializationSettings('@drawable/ic_stat_notify');
  await local.initialize(
    const InitializationSettings(android: android),
  );

  final androidPlugin = local.resolvePlatformSpecificImplementation<
      AndroidFlutterLocalNotificationsPlugin>();
  await androidPlugin?.requestNotificationsPermission();
}

Future<void> showForegroundMessage(RemoteMessage message) async {
  final notification = message.notification;
  if (notification == null) return;

  await local.show(
    message.hashCode,
    notification.title,
    notification.body,
    const NotificationDetails(
      android: AndroidNotificationDetails(
        'alerts',
        '서비스 알림',
        channelDescription: '서비스 상태 알림',
        importance: Importance.high,
        priority: Priority.high,
      ),
    ),
  );
}
```

아이콘은 런처 아이콘보다 `android/app/src/main/res/drawable`에 둔 흰색 단색 알림 아이콘을 쓰는 편이 안전하다. `alerts` 채널을 이미 사용자가 낮은 중요도로 만든 뒤 코드에서 `Importance.high`로 바꿔도 기존 채널 설정은 자동으로 올라가지 않는다. 테스트 기기에서 채널을 삭제하거나 새 채널 ID로 검증해야 한다.

## 출시 전 체크리스트

1. `compileSdk`와 `targetSdk`가 최소 33 이상인지 확인한다.
2. Android 13 실기기에서 최초 실행 후 알림 허용 대화상자가 실제로 뜨는지 확인한다.
3. 허용, 거부, 설정에서 다시 허용한 세 상태를 각각 확인한다.
4. FCM 포그라운드·백그라운드·종료 상태를 나눠 전송한다.
5. 포그라운드에서는 `onMessage`와 로컬 알림 표시 로그를 따로 남긴다.
6. `adb shell pm revoke 패키지명 android.permission.POST_NOTIFICATIONS`로 재설치 없이 거부 상태를 재현한다.
7. 알림 채널의 소리·중요도는 새 설치와 기존 설치에서 각각 확인한다.

Android 12L 이하를 계속 지원하면서 `targetSdk`가 32 이하라면 첫 채널 생성 시 시스템이 권한 대화상자를 띄우는 시점이 달라진다. 백그라운드에서 FCM이 처음 채널을 만들면 대화상자가 즉시 나오지 않고 알림이 누락될 수 있으므로, 가능하면 target SDK를 33 이상으로 올려 앱 흐름 안에서 명시적으로 권한을 요청하는 편이 낫다.

핵심은 “FCM 권한”, “포그라운드 표시”, “알림 채널”을 하나의 성공 조건으로 뭉개지 않는 것이다. 출시 빌드에서 알림이 안 보이면 권한 상태 → 앱 상태 → 채널 설정 순서로 분리하면 재현 범위를 빠르게 좁힐 수 있다.

참고 문서: [Firebase Flutter 메시지 수신 가이드](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages), [Android 알림 런타임 권한](https://developer.android.com/develop/ui/compose/notifications/notification-permission), [flutter_local_notifications Android 설정](https://pub.dev/packages/flutter_local_notifications)
