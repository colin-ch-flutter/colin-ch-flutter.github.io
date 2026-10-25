---
layout: post
title: "Flutter Android 알림 권한 오류 - FCM 첫 채널이 백그라운드에서 만들어질 때"
description: "Flutter Android 13 이상에서 FCM 알림 권한 대화상자가 뜨지 않거나 첫 알림이 사라질 때, target SDK·notification channel·release Manifest를 구분해 복구하는 방법을 정리한다."
date: 2026-10-26
tags: [Flutter, Android, Firebase, FCM, release]
comments: true
share: true
---

![Flutter Android FCM 알림 권한과 release 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

이 문제는 Android 13(API 33) 이상을 지원하면서 FCM을 처음 붙이는 앱이 우선 확인해야 한다. 앱을 새로 설치한 뒤 백그라운드 FCM만 기다리는 구조라면 첫 알림이 표시되지 않고 권한 창도 나중에 뜰 수 있다. 반대로 앱이 알림을 전혀 보내지 않거나 Android 12 이하만 지원한다면 `POST_NOTIFICATIONS` 흐름을 억지로 추가할 필요는 없다.

## 증상은 같아도 원인은 세 가지다

Android 13부터 새로 설치한 앱의 알림은 기본 차단된다. target SDK가 33 이상이면 앱이 권한 요청 시점을 정하고, 32 이하이면 시스템이 첫 Activity 실행과 notification channel 생성 시점을 조합해 대화상자를 띄운다. Firebase 문서가 따로 경고하는 조건은 target SDK 32 이하 앱에서 첫 channel이 백그라운드 FCM 수신 중 만들어지는 경우다. 이때 알림은 표시되지 않고 권한 요청도 앱을 다시 열 때까지 지연된다.

| 확인 대상 | 잘못된 가정 | 실제 점검 |
|---|---|---|
| Android 13+ 신규 설치 | FCM이 권한 창을 알아서 띄운다 | `POST_NOTIFICATIONS` 요청 시점을 앱이 제어하는지 확인 |
| 첫 notification channel | 서버가 메시지를 보내면 channel도 준비된다 | 포그라운드에서 channel을 만든 뒤 권한을 요청 |
| release 빌드 | debug Manifest 권한이 그대로 들어간다 | merged Manifest와 실제 설치 앱의 알림 설정 확인 |

## 포그라운드에서 권한과 channel을 준비한다

권한 요청과 채널 생성을 앱이 알림 기능을 설명할 수 있는 화면에서 실행한다. 아래 예시는 `firebase_messaging`과 `flutter_local_notifications`를 함께 쓰는 구성이다. 앱 시작 직후 무조건 팝업을 띄우기보다 사용자가 “알림 받기”를 선택했을 때 호출하는 편이 이탈을 줄인다.

```dart
final _channel = const AndroidNotificationChannel(
  'alerts_high',
  '중요 알림',
  description: '기기 상태와 장애 알림을 표시한다.',
  importance: Importance.high,
);

Future<AuthorizationStatus> prepareNotifications() async {
  final messaging = FirebaseMessaging.instance;
  final local = FlutterLocalNotificationsPlugin();

  await local.initialize(
    const InitializationSettings(
      android: AndroidInitializationSettings('@mipmap/ic_launcher'),
    ),
  );
  await local
      .resolvePlatformSpecificImplementation<
          AndroidFlutterLocalNotificationsPlugin>()
      ?.createNotificationChannel(_channel);

  final settings = await messaging.requestPermission(
    alert: true,
    badge: true,
    sound: true,
  );
  return settings.authorizationStatus;
}
```

이 코드의 순서는 채널을 먼저 만들고 권한을 요청하는 데 의미가 있다. FCM notification payload가 오기 전에 Android가 표시할 채널을 알고 있어야 하며, 서버에서 첫 메시지를 보내 채널 생성을 대신하게 두면 백그라운드·종료 상태에서 타이밍이 꼬인다. FCM payload의 `android.notification.channel_id`도 `alerts_high`처럼 실제 생성한 ID와 같아야 한다.

## release에서 확인할 값

`android/app/src/main/AndroidManifest.xml`에 아래 권한이 병합되는지 확인한다. 플러그인이 manifest에 선언해 주는 버전도 있지만, release 결과를 기준으로 검사해야 한다.

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

```bash
flutter build appbundle --release
apkanalyzer manifest permissions \
  build/app/outputs/bundle/release/app-release.aab \
  | grep POST_NOTIFICATIONS
```

명령 결과가 비어 있으면 Gradle manifest 병합, flavor별 `src/<flavor>/AndroidManifest.xml`, 플러그인 버전을 확인한다. 결과가 있어도 이전에 “허용 안 함”을 선택한 기기에서는 창이 다시 뜨지 않을 수 있다. 테스트마다 앱을 삭제하거나 설정에서 알림 권한을 초기화하고, Android 13 이상 신규 설치·Android 12 기존 설치·백그라운드 첫 수신을 나눠 검증해야 한다.

## 짧은 출시 체크리스트

- [ ] Android 13 이상에서 `requestPermission()` 호출 결과를 저장했다.
- [ ] 첫 FCM 수신 전에 포그라운드에서 channel을 생성했다.
- [ ] 서버 payload의 `channel_id`와 앱 channel ID가 같다.
- [ ] release AAB의 merged Manifest에 `POST_NOTIFICATIONS`가 있다.
- [ ] 권한 거부·알림 설정 변경·앱 재설치 상태를 각각 테스트했다.

핵심은 FCM 전송 성공과 Android 화면 표시를 같은 단계로 보지 않는 것이다. Firebase 콘솔에서 발송 성공이어도 OS 권한이 없거나 첫 channel이 백그라운드에서 생성됐다면 사용자는 알림을 보지 못한다. 2026년 10월 26일 확인 기준으로 [Firebase의 Android 알림 권한 안내](https://firebase.google.com/docs/cloud-messaging/android/get-started)는 이 백그라운드 channel 조건을 명시하고, [Android의 알림 런타임 권한 문서](https://developer.android.com/develop/ui/views/notifications/notification-permission)는 target SDK별 대화상자 시점을 구분한다.
