---
layout: post
title: "flutter_local_notifications로 로컬 알림 구현"
description: " "
date: 2026-05-31
tags: [Flutter, LocalNotification, 알림, flutter_local_notifications]
comments: true
share: true
---

# flutter_local_notifications로 로컬 알림 구현

![알림 화면](https://images.unsplash.com/photo-1614064641938-3bbee52942c7?w=800&q=80)

## 로컬 알림이 필요한 경우

FCM으로 서버에서 알림을 보내는 것과 달리, 로컬 알림은 앱 자체에서 발생시키는 알림이다. 이런 경우에 필요하다:

- 포그라운드에서 FCM 메시지를 수신하고 알림으로 표시
- 앱 내에서 설정한 예약 알림 (예: "30분 후 IoT 기기 켜기 알림")
- 스케줄 실행 전 미리 알림

## 초기화

```dart
class LocalNotificationService {
  final FlutterLocalNotificationsPlugin _plugin =
      FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: false, // FCM에서 이미 요청
      requestBadgePermission: false,
      requestSoundPermission: false,
    );

    await _plugin.initialize(
      const InitializationSettings(
        android: androidSettings,
        iOS: iosSettings,
      ),
      onDidReceiveNotificationResponse: _onNotificationTapped,
      onDidReceiveBackgroundNotificationResponse: _onBackgroundNotificationTapped,
    );

    // Android 알림 채널 생성
    await _createChannels();
  }

  Future<void> _createChannels() async {
    const channels = [
      AndroidNotificationChannel(
        'iot_device_alerts',
        'IoT 기기 알림',
        description: '기기 이상, 오류 알림',
        importance: Importance.high,
      ),
      AndroidNotificationChannel(
        'iot_device_info',
        'IoT 기기 정보',
        description: '예열 완료, 모드 변경 등 일반 알림',
        importance: Importance.defaultImportance,
      ),
    ];

    final androidPlugin = _plugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>();
    
    for (final channel in channels) {
      await androidPlugin?.createNotificationChannel(channel);
    }
  }
}
```

## 즉시 알림 표시

```dart
Future<void> showNotification({
  required int id,
  required String title,
  required String body,
  String channelId = 'iot_device_info',
  String? payload,
}) async {
  final androidDetails = AndroidNotificationDetails(
    channelId,
    channelId == 'iot_device_alerts' ? 'IoT 기기 알림' : 'IoT 기기 정보',
    importance: channelId == 'iot_device_alerts'
        ? Importance.high
        : Importance.defaultImportance,
    priority: Priority.high,
    showWhen: true,
    icon: '@drawable/ic_notification',
  );

  final iosDetails = DarwinNotificationDetails(
    presentAlert: true,
    presentBadge: true,
    presentSound: true,
    threadIdentifier: channelId,
  );

  await _plugin.show(
    id,
    title,
    body,
    NotificationDetails(android: androidDetails, iOS: iosDetails),
    payload: payload,
  );
}
```

## 예약 알림

IoT 기기 스케줄이 실행되기 전에 미리 알림을 보내는 경우:

```dart
Future<void> scheduleNotification({
  required int id,
  required String title,
  required String body,
  required DateTime scheduledDate,
  String? payload,
}) async {
  await _plugin.zonedSchedule(
    id,
    title,
    body,
    tz.TZDateTime.from(scheduledDate, tz.local),
    NotificationDetails(
      android: const AndroidNotificationDetails(
        'iot_device_schedule',
        '스케줄 알림',
      ),
    ),
    androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
    uiLocalNotificationDateInterpretation:
        UILocalNotificationDateInterpretation.absoluteTime,
    payload: payload,
  );
}
```

예약 알림에는 timezone 패키지가 필요하다:

```dart
// 앱 시작 시 타임존 초기화
import 'package:timezone/data/latest.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

void initializeTimezone() {
  tz.initializeTimeZones();
  final localTimezone = await FlutterTimezone.getLocalTimezone();
  tz.setLocalLocation(tz.getLocation(localTimezone));
}
```

## 알림 탭 처리

알림을 탭했을 때 특정 화면으로 이동:

```dart
void _onNotificationTapped(NotificationResponse response) {
  final payload = response.payload;
  if (payload == null) return;

  try {
    final data = jsonDecode(payload) as Map<String, dynamic>;
    final type = data['type'] as String?;

    switch (type) {
      case 'iot_device_error':
        Get.toNamed(Routes.iot_deviceControl, arguments: data);
        break;
      case 'schedule_reminder':
        Get.toNamed(Routes.iot_deviceSchedule);
        break;
      default:
        Get.toNamed(Routes.notificationList);
    }
  } catch (_) {
    Get.toNamed(Routes.notificationList);
  }
}

// 백그라운드용 (top-level 함수)
@pragma('vm:entry-point')
void _onBackgroundNotificationTapped(NotificationResponse response) {
  // 백그라운드에서는 제한적으로만 처리 가능
}
```

---

다음 편은 알림 설정 화면 구현이다.
