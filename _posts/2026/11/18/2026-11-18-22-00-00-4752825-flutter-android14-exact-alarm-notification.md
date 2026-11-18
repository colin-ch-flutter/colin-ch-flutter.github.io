---
layout: post
title: "Flutter Android 14 예약 알림 오류 - exact alarm 권한과 inexact 대체 경로"
description: "Flutter flutter_local_notifications로 Android 14 예약 알림이 출시 후 사라질 때 SCHEDULE_EXACT_ALARM, 알림 권한, 부정확 알림 대체 경로를 점검하는 방법을 정리한다."
date: 2026-11-18
tags: [Flutter, Android, 배포, 알림, flutter_local_notifications]
comments: true
share: true
---

![Flutter Android 14 예약 알림과 exact alarm 권한을 점검하는 개발 화면](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Android 14 이상에서 Flutter 예약 알림이 release 앱에서만 누락된다면 `flutter_local_notifications`의 코드보다 exact alarm 권한을 먼저 확인해야 한다. 약속 시간에 반드시 울려야 하는 알람 앱은 `SCHEDULE_EXACT_ALARM`을 확인하고 사용자에게 설정 화면을 안내해야 한다. 반대로 단순 리마인더나 동기화 알림이라면 exact 권한을 요구하지 않고 `inexactAllowWhileIdle`로 낮추는 편이 출시 심사와 사용 경험에 유리하다. 알림 자체를 사용하지 않는 앱에는 이 대응이 필요하지 않다.

## Android 14에서 갑자기 예약이 사라지는 이유

Android 14는 Android 13 이상을 대상으로 새로 설치한 앱의 `SCHEDULE_EXACT_ALARM`을 기본 허용하지 않는다. 권한이 없는 상태에서 `exact` 또는 `exactAllowWhileIdle` 모드로 예약하면 플러그인이 로그를 남기거나 예약을 만들지 못한다. 앱 업데이트에서는 기존 권한이 유지될 수 있지만, 새 기기로 백업 복원한 경우에는 다시 거부될 수 있다.

여기서 권한을 두 개로 나눠야 한다.

| 문제 | 확인할 권한 | 영향 |
| --- | --- | --- |
| 알림을 표시할 수 없음 | `POST_NOTIFICATIONS` | Android 13 이상에서 알림 표시 차단 |
| 정확한 시각에 예약할 수 없음 | `SCHEDULE_EXACT_ALARM` | exact 예약 실패 또는 취소 |
| 대략적인 시각이면 충분함 | 없음 | inexact 예약으로 배터리·권한 요구 감소 |

Android 공식 문서 기준으로 `USE_EXACT_ALARM`은 설치 시 자동 허용되지만 알람·캘린더처럼 제한된 용도와 Play 정책을 전제로 한다. 일반적인 습관 알림이나 IoT 상태 알림에 이 권한을 무조건 추가하는 선택은 피하는 게 안전하다.

## Manifest는 예약 기능을 쓸 때만 추가한다

예약 알림과 부팅 후 재예약을 사용하는 앱이라면 `android/app/src/main/AndroidManifest.xml`의 `<manifest>` 안에 다음 권한을 둔다. `POST_NOTIFICATIONS`는 Android 13 이상에서 별도로 런타임 요청해야 한다.

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
```

`SCHEDULE_EXACT_ALARM`을 선언했다고 허용된 것은 아니다. 사용자가 앱의 기능을 선택한 시점에 권한 상태를 확인하고, 필요할 때만 시스템 설정으로 보낸다. 앱 시작 직후 권한 팝업을 띄우면 알림이 왜 필요한지 설명하기 어렵고 거부율도 높아진다.

## Flutter에서 exact와 fallback을 분리한다

예약 시점에 권한을 확인하지 않고 무조건 exact 모드를 넘기는 것이 흔한 실패 원인이다. 아래 코드는 exact 권한이 있으면 정확한 모드를 사용하고, 없으면 사용자가 선택한 기능에 따라 설정 화면을 안내하거나 inexact 모드로 예약한다.

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

final notifications = FlutterLocalNotificationsPlugin();

Future<bool> ensureExactAlarmAccess() async {
  final android = notifications.resolvePlatformSpecificImplementation<
      AndroidFlutterLocalNotificationsPlugin>();

  if (android == null) return false;

  final canSchedule = await android.canScheduleExactNotifications() ?? false;
  if (canSchedule) return true;

  // 사용자가 '정확한 알림'을 선택한 화면에서만 호출한다.
  await android.requestExactAlarmsPermission();
  return await android.canScheduleExactNotifications() ?? false;
}

Future<void> scheduleReminder({required bool mustRingOnTime}) async {
  final exact = mustRingOnTime && await ensureExactAlarmAccess();

  await notifications.zonedSchedule(
    1001,
    '예약 알림',
    '설정한 시간이 되었다.',
    nextScheduleTime(),
    const NotificationDetails(
      android: AndroidNotificationDetails(
        'reminder',
        '예약 알림',
        channelDescription: '예약된 알림을 표시한다.',
      ),
    ),
    androidScheduleMode: exact
        ? AndroidScheduleMode.exactAllowWhileIdle
        : AndroidScheduleMode.inexactAllowWhileIdle,
    uiLocalNotificationDateInterpretation:
        UILocalNotificationDateInterpretation.absoluteTime,
  );
}
```

위 예제의 `nextScheduleTime()`은 앱의 시간대 정책에 맞는 `tz.TZDateTime`을 반환해야 한다. 시간대 초기화를 생략하면 권한 문제를 고친 뒤에도 해외 이동이나 서머타임에서 알림 시각이 어긋날 수 있다. 또 `mustRingOnTime`을 단순히 사용자가 켠 “알림 받기” 옵션과 섞지 말고, 정확도가 필요한 기능인지 별도 상태로 저장해야 한다.

정확한 알림이 아닌 경우에는 inexact가 무료 대안이다. Android는 배터리 절약을 위해 실행 시각을 조정할 수 있으므로 “오전 9시 전후에 동기화 결과를 알려주는” 기능에는 충분하지만, 복약·예약 시작처럼 1분 오차도 허용하지 않는 기능에는 적합하지 않다.

## 출시 전에 재현할 체크리스트

1. Android 14 이상 새 설치 상태에서 알림 권한과 exact alarm 접근을 모두 거부한다.
2. `mustRingOnTime: true`인 기능은 설정 화면으로 이동하고, 복귀 후 권한 상태를 다시 읽는다.
3. 권한을 허용하지 않은 상태에서 예약이 예외로 종료되지 않고 inexact 또는 안내 상태로 남는지 확인한다.
4. 기기를 재부팅한 뒤 `RECEIVE_BOOT_COMPLETED`와 플러그인의 재예약 설정을 확인한다.
5. Play App Signing으로 만든 release APK에서 알림 채널, 작은 아이콘, 예약 목록을 확인한다.

특히 설정 화면에서 돌아온 뒤 기존 `canScheduleExactNotifications()` 값을 캐시하면 안 된다. Android 공식 흐름도 `onResume` 시점에 권한을 다시 검사하고, 허용되지 않았으면 대체 경로를 제공하라고 안내한다. 테스트는 설정에서 “알람 및 리마인더” 접근을 끈 뒤 예약하는 방식으로 재현할 수 있다.

정리하면 `POST_NOTIFICATIONS`는 표시 권한이고, `SCHEDULE_EXACT_ALARM`은 정확한 예약 권한이다. 두 상태를 분리해 확인하고, exact가 제품 요구사항이 아닐 때는 inexact를 기본값으로 두면 Android 14 출시 후 조용히 사라지는 예약 알림을 줄일 수 있다.

참고 문서: [Android exact alarm 공식 가이드](https://developer.android.com/about/versions/14/changes/schedule-exact-alarms), [Android 알람 예약 가이드](https://developer.android.com/develop/background-work/services/alarms), [flutter_local_notifications Android 설정](https://pub.dev/packages/flutter_local_notifications)
