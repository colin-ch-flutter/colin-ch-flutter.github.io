---
layout: post
title: "알림 설정 화면 - 사용자가 원하는 알림만 받기"
description: " "
date: 2026-06-01
tags: [Flutter, 알림설정, UX, FCM, 푸시알림]
comments: true
share: true
---

# 알림 설정 화면 - 사용자가 원하는 알림만 받기

![알림 설정](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

## 알림이 너무 많으면 오히려 방해가 된다

IoT 앱은 기기에서 오는 이벤트가 많다. 기기가 켜질 때, 꺼질 때, 온도 도달, 오류 발생, 펌웨어 업데이트... 이 모든 걸 알림으로 보내면 사용자는 결국 알림을 끈다.

그래서 알림 종류별로 ON/OFF를 선택할 수 있는 설정 화면이 필요하다.

## 알림 종류 정의

```dart
// domain/models/notification_setting.dart
enum NotificationType {
  deviceError,        // 기기 오류
  deviceOffline,      // 기기 오프라인
  scheduleExecuted,   // 스케줄 실행
  fotaAvailable,      // 펌웨어 업데이트 가능
  fotaCompleted,      // 펌웨어 업데이트 완료
  temperatureReached, // 목표 온도 도달
}

class NotificationSetting {
  final NotificationType type;
  final bool isEnabled;
  
  const NotificationSetting({required this.type, required this.isEnabled});
}
```

## 알림 설정 화면 구현

```dart
class NotificationSettingsView extends GetView<NotificationSettingsController> {
  const NotificationSettingsView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('알림 설정')),
      body: Obx(() {
        return ListView(
          children: [
            _SectionHeader(title: '기기 알림'),
            _NotificationTile(
              title: '기기 오류',
              subtitle: 'IoT 기기에 오류가 발생하면 알림을 받습니다',
              icon: Icons.error_outline,
              isEnabled: controller.isEnabled(NotificationType.deviceError),
              onChanged: (v) => controller.setEnabled(NotificationType.deviceError, v),
            ),
            _NotificationTile(
              title: '기기 오프라인',
              subtitle: '기기가 인터넷 연결을 잃으면 알림을 받습니다',
              icon: Icons.wifi_off,
              isEnabled: controller.isEnabled(NotificationType.deviceOffline),
              onChanged: (v) => controller.setEnabled(NotificationType.deviceOffline, v),
            ),
            _SectionHeader(title: '스케줄/자동화'),
            _NotificationTile(
              title: '스케줄 실행',
              subtitle: '예약된 스케줄이 실행되면 알림을 받습니다',
              icon: Icons.schedule,
              isEnabled: controller.isEnabled(NotificationType.scheduleExecuted),
              onChanged: (v) => controller.setEnabled(NotificationType.scheduleExecuted, v),
            ),
            _SectionHeader(title: '업데이트'),
            _NotificationTile(
              title: '펌웨어 업데이트',
              subtitle: '새 펌웨어가 있으면 알림을 받습니다',
              icon: Icons.system_update,
              isEnabled: controller.isEnabled(NotificationType.fotaAvailable),
              onChanged: (v) => controller.setEnabled(NotificationType.fotaAvailable, v),
            ),
          ],
        );
      }),
    );
  }
}
```

## 설정 저장 및 서버 동기화

```dart
class NotificationSettingsController extends GetxController {
  final ApiRepository _apiRepo;
  final RxMap<NotificationType, bool> _settings = 
      <NotificationType, bool>{}.obs;

  bool isEnabled(NotificationType type) {
    return _settings[type] ?? true; // 기본값 활성
  }

  Future<void> setEnabled(NotificationType type, bool value) async {
    _settings[type] = value;
    
    // 로컬 저장
    _saveToLocal();
    
    // 서버 동기화
    await _apiRepo.updateNotificationSettings(
      NotificationSetting(type: type, isEnabled: value),
    );
    
    // FCM 토픽 구독/해제 (서버 측 필터링이 어려운 경우)
    if (value) {
      await FirebaseMessaging.instance.subscribeToTopic(type.topicName);
    } else {
      await FirebaseMessaging.instance.unsubscribeFromTopic(type.topicName);
    }
  }
}
```

## 알림 내역 화면

수신된 알림 기록을 보여주는 화면도 필요하다. `NotificationListView`와 `NotificationDetailView`가 이를 담당한다.

```dart
class NotificationListView extends GetView<NotificationListController> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('알림'),
        actions: [
          TextButton(
            onPressed: controller.markAllAsRead,
            child: const Text('모두 읽음'),
          ),
        ],
      ),
      body: Obx(() {
        if (controller.notifications.isEmpty) {
          return const Center(child: Text('알림이 없습니다'));
        }
        
        return ListView.builder(
          itemCount: controller.notifications.length,
          itemBuilder: (context, index) {
            final notif = controller.notifications[index];
            return ListTile(
              leading: Icon(
                _iconForType(notif.type),
                color: notif.isRead ? Colors.grey : Theme.of(context).colorScheme.primary,
              ),
              title: Text(
                notif.title,
                style: TextStyle(
                  fontWeight: notif.isRead ? FontWeight.normal : FontWeight.bold,
                ),
              ),
              subtitle: Text(notif.body),
              trailing: Text(
                _formatTime(notif.createdAt),
                style: const TextStyle(fontSize: 12, color: Colors.grey),
              ),
              onTap: () => controller.openNotification(notif),
            );
          },
        );
      }),
    );
  }
}
```

---

다음 편부터는 사용자 관리 파트다. 회원가입 플로우부터 시작한다.
