---
layout: post
title: "FCM 푸시 알림 완전 정복 - 포그라운드/백그라운드/종료 상태 모두 처리"
description: " "
date: 2026-05-30
tags: [Flutter, FCM, 푸시알림, Firebase, 알림]
comments: true
share: true
---
![푸시 알림](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

## 앱 상태 세 가지를 모두 처리해야 한다

FCM 구현에서 가장 헷갈리는 부분이 바로 앱 상태에 따른 처리다. 세 가지 상태가 있다:

1. **Foreground**: 앱이 실행 중이고 화면이 보이는 상태
2. **Background**: 앱이 백그라운드에 있는 상태 (홈 버튼 눌러서)
3. **Terminated**: 앱이 완전히 종료된 상태 (스와이프로 닫거나 기기 재시작 후)

각 상태에서 알림이 오는 방식과 처리 방법이 다르다.

## main.dart 설정

```dart
// 백그라운드 핸들러는 반드시 top-level 함수여야 한다
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // 백그라운드에서는 Firebase가 초기화되지 않아서 다시 초기화 필요
  await Firebase.initializeApp();
  
  // 여기서는 UI 작업 불가 (앱이 실제로 실행 중이 아님)
  // 데이터 저장 정도만 가능
  print('백그라운드 메시지: ${message.data}');
}

void main() async {
  await Firebase.initializeApp();
  
  // 반드시 Firebase 초기화 후에 등록
  FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
  
  // ...
}
```

## PushNotificationService 구현

```dart
// core/services/push_notification_service.dart
class PushNotificationService {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> init() async {
    // iOS 권한 요청
    await _requestPermission();
    
    // FCM 토큰 획득 및 서버 등록
    await _registerToken();
    
    // 포그라운드 메시지 처리
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);
    
    // 백그라운드 → 앱 실행 (알림 탭)
    FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);
    
    // Terminated → 앱 실행 (알림 탭으로 앱 켜짐)
    final initialMessage = await _messaging.getInitialMessage();
    if (initialMessage != null) {
      _handleNotificationTap(initialMessage);
    }
  }

  Future<void> _requestPermission() async {
    final settings = await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
      provisional: false,
    );
    print('알림 권한: ${settings.authorizationStatus}');
  }

  Future<void> _registerToken() async {
    final token = await _messaging.getToken();
    if (token != null) {
      // 서버에 토큰 등록
      await _apiRepo.registerFcmToken(token);
    }
    
    // 토큰 갱신 감지
    _messaging.onTokenRefresh.listen((newToken) {
      _apiRepo.registerFcmToken(newToken);
    });
  }

  void _handleForegroundMessage(RemoteMessage message) {
    // 포그라운드에서는 FCM이 자동으로 알림을 표시하지 않음
    // flutter_local_notifications로 직접 표시해야 함
    _showLocalNotification(message);
  }

  void _handleNotificationTap(RemoteMessage message) {
    // 알림 데이터에서 이동할 화면 결정
    final notificationType = message.data['type'];
    
    switch (notificationType) {
      case 'iot_device_alert':
        final deviceId = message.data['deviceId'];
        Get.toNamed(Routes.iot_deviceControl, arguments: {'deviceId': deviceId});
        break;
      case 'fota_available':
        Get.toNamed(Routes.iot_deviceScadaFota);
        break;
      default:
        Get.toNamed(Routes.notificationList);
    }
  }
}
```

## 포그라운드 알림 표시

포그라운드에서는 FCM이 알림을 자동으로 보여주지 않는다. `flutter_local_notifications`를 써서 직접 표시한다:

```dart
void _showLocalNotification(RemoteMessage message) {
  final notification = message.notification;
  if (notification == null) return;

  _localNotifications.show(
    message.hashCode,
    notification.title,
    notification.body,
    NotificationDetails(
      android: AndroidNotificationDetails(
        'iot_device_alerts',
        'IoT 기기 알림',
        importance: Importance.high,
        priority: Priority.high,
      ),
      iOS: const DarwinNotificationDetails(
        presentAlert: true,
        presentBadge: true,
        presentSound: true,
      ),
    ),
    payload: jsonEncode(message.data),
  );
}
```

## Android 알림 채널

Android 8.0(API 26) 이상에서는 알림 채널이 필요하다:

```dart
// 앱 시작 시 채널 생성
await _localNotifications
    .resolvePlatformSpecificImplementation<
        AndroidFlutterLocalNotificationsPlugin>()
    ?.createNotificationChannel(
  const AndroidNotificationChannel(
    'iot_device_alerts',
    'IoT 기기 알림',
    description: 'IoT 기기 기기 관련 알림',
    importance: Importance.high,
  ),
);
```

## iOS 포그라운드 알림 설정

iOS는 기본적으로 포그라운드에서 알림이 표시되지 않는다. 직접 설정해줘야 한다:

```dart
await _messaging.setForegroundNotificationPresentationOptions(
  alert: true,
  badge: true,
  sound: true,
);
```

---

다음 편은 flutter_local_notifications 심화 활용이다.
