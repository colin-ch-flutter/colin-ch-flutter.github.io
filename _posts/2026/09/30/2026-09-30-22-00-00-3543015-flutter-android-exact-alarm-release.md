---
layout: post
title: "Flutter Android 예약 알림 오류 - Android 14+ SCHEDULE_EXACT_ALARM 출시 대응"
description: "Flutter 예약 알림이 Android 14 출시본에서 조용히 실패할 때 SCHEDULE_EXACT_ALARM 권한과 부정확 알람 전환 기준을 정리한다."
date: 2026-09-30
tags: [Flutter, Android, Firebase, 배포·운영]
comments: true
share: true
---

![Flutter Android 예약 알림과 exact alarm 권한 분기](/images/2026-09-30-flutter-android-exact-alarm.png)

Android 14 이상에서 Flutter 예약 알림이 실행되지 않는다면 알림 권한만 확인해서는 부족하다. `flutter_local_notifications`로 사용자가 정한 시각에 반드시 울려야 하는 기능을 만들었다면 `SCHEDULE_EXACT_ALARM` 상태를 확인하고, 필요할 때만 설정 화면으로 보내야 한다. 일정 알림이 몇 분 늦어도 되는 앱이라면 exact alarm 권한을 추가하지 않는 쪽이 출시와 배터리 운영에 유리하다.

## 왜 debug에서는 보이고 출시본에서만 빠지는가

Android 12(API 31)부터 target SDK가 높은 앱은 정확한 알람을 사용하기 전에 special app access를 확인해야 한다. Android 14에서는 Android 13(API 33) 이상을 대상으로 새로 설치한 대부분의 앱에 `SCHEDULE_EXACT_ALARM`이 기본 허용되지 않는다. 알림의 `POST_NOTIFICATIONS`를 허용했어도 별개의 권한이라 예약 시각이 되면 조용히 누락될 수 있다.

| 기능 조건 | 권한·API 선택 | 출시 판단 |
|---|---|---|
| 알람시계, 복약처럼 시각 오차가 치명적 | `SCHEDULE_EXACT_ALARM` + 상태 확인 | 사용자에게 필요성을 설명하고 설정 화면으로 보낸다 |
| 캘린더·알람시계가 핵심 기능 | `USE_EXACT_ALARM` 검토 | Google Play 정책상 허용된 앱 유형인지 확인한다 |
| 하루 중 몇 분 차이는 허용 | inexact alarm | exact 권한을 선언하지 않고 배터리 부담을 줄인다 |
| 서버에서 보내는 일반 푸시 | FCM 등 서버 전달 | 로컬 exact alarm 문제와 분리해서 진단한다 |

Android 공식 문서 기준으로 `SCHEDULE_EXACT_ALARM`은 사용자가 허용할 수 있고 시스템이나 사용자가 다시 철회할 수 있다. 백업 복원으로 Android 14 기기에 옮긴 앱도 권한이 거부된 상태가 될 수 있다. 한 번 허용된 적이 있다는 이유로 앱 시작 때 상태를 캐시하면 안 되는 이유다.

## Manifest에 권한을 넣는 위치

정확한 예약 알림을 정말 지원할 때만 앱 모듈의 `android/app/src/main/AndroidManifest.xml`에 선언한다.

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />

    <application
        android:label="my_app"
        android:name="${applicationName}">
        <!-- flutter_local_notifications가 요구하는 receiver 설정은
             고정한 패키지 버전의 README와 함께 확인한다. -->
    </application>
</manifest>
```

`USE_EXACT_ALARM`으로 바꾸면 사용자 설정 화면을 거치지 않아도 되지만, 모든 리마인더 앱에 쓸 수 있는 우회 권한은 아니다. 정확한 알람이 앱의 핵심 기능인지와 Play Console 정책을 함께 확인해야 한다.

## Flutter에서 예약 전에 상태를 확인한다

권한 요청을 앱 첫 화면 진입 때 무조건 띄우면 사용자는 기능과 무관한 시스템 설정으로 이동하게 된다. 사용자가 “정시 알림 사용”을 켰을 때 검사하고, 거부하면 대체 경로를 명확히 보여주는 편이 낫다.

```dart
final notifications = FlutterLocalNotificationsPlugin();

Future<bool> ensureExactAlarmAccess() async {
  final android = notifications.resolvePlatformSpecificImplementation<
      AndroidFlutterLocalNotificationsPlugin>();

  if (android == null) return false;

  final canSchedule = await android.canScheduleExactAlarms() ?? false;
  if (canSchedule) return true;

  // 앱 시작 직후가 아니라 사용자가 '정시 알림'을 선택한 시점에 호출한다.
  final granted = await android.requestExactAlarmsPermission() ?? false;
  return granted;
}
```

위 함수가 `false`를 반환한 상태에서 `zonedSchedule` 같은 예약 API를 그대로 호출하지 않는 것이 핵심이다. 정시성이 필수가 아니면 `androidScheduleMode`를 부정확 알람 모드로 바꾸고, 필수라면 “알람 및 리마인더 허용” 화면으로 이동해야 한다. 사용하는 `flutter_local_notifications` 버전에 따라 초기화와 예약 메서드의 이름·enum이 달라질 수 있으므로, 코드 복사보다 고정한 버전의 README를 기준으로 맞춘다.

## 출시 전 재현 체크리스트

- [ ] Android 13/14/15/16에서 `POST_NOTIFICATIONS`와 exact alarm을 별도로 확인했다.
- [ ] 새 설치와 백업 복원 상태를 각각 테스트했다.
- [ ] 설정에서 exact alarm을 철회한 뒤 앱을 다시 열어 상태를 재조회했다.
- [ ] 권한 거부 시 예약을 강행하지 않고 부정확 알람 또는 사용자 안내로 분기한다.
- [ ] 기기 재부팅·시간대 변경·앱 업데이트 뒤 예약을 복원하는지 확인했다.
- [ ] Play Console에 올릴 AAB의 Manifest에 불필요한 exact alarm 권한이 남지 않았는지 확인했다.

예약 알림이 몇 분 늦어도 되는 서비스라면 이 문제를 권한 요청으로 해결할 필요가 없다. Android 공식 가이드도 사용자에게 보이는 시간 민감 기능에만 exact alarm을 쓰고, 일반 작업은 inexact alarm이나 WorkManager를 고려하라고 안내한다. 반대로 복약·알람처럼 시각 보장이 제품 약속이라면 권한 상태를 매번 확인하고, 철회·복원까지 포함해 출시 승인 조건으로 관리해야 한다.

참고 기준일은 2026년 9월 30일이다. 세부 동작은 Android 버전과 고정한 플러그인 버전에 따라 달라질 수 있다.

- [Android exact alarm 권한 변경 안내](https://developer.android.com/about/versions/14/changes/schedule-exact-alarms)
- [Android 알람 예약 가이드](https://developer.android.com/develop/background-work/services/alarms)
- [Android AlarmManager API](https://developer.android.com/reference/android/app/AlarmManager)
- [flutter_local_notifications Android 설정](https://github.com/MaikuB/flutter_local_notifications#androidmanifestxml-setup)
