---
layout: post
title: "flutter_local_notifications 단위 테스트 — FakeNotificationService로 네이티브 없이 격리"
description: "flutter_local_notifications는 네이티브 의존성 때문에 유닛 테스트에서 직접 쓸 수가 없다. 인터페이스 추상화와 FakeNotificationService 패턴으로 IoT 앱의 알림 로직을 네이티브 없이 격리 테스트하는 방법을 실제 코드로 정리했다."
date: 2026-06-28
tags: [Flutter, flutter_local_notifications, IoT, CleanArchitecture, 스마트홈]
comments: true
share: true
---

![스마트홈 IoT 알림 테스트](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

`flutter_local_notifications`를 유닛 테스트에서 그냥 호출하면 `MissingPluginException`이 터진다. 알림 로직을 테스트하려면 네이티브를 걷어내고 인터페이스 뒤로 숨겨야 한다. 인터페이스 추상화와 `FakeNotificationService`만 있으면 CI에서도, 시뮬레이터에서도 알림 로직을 완전히 격리해서 테스트할 수 있다.

## 왜 알림 테스트가 까다로운가

IoT 앱에서 알림은 핵심이다. 보일러 온도가 임계값을 넘으면 알림, 기기 연결이 끊기면 알림, 예약 시간이 되면 알림. 근데 이걸 테스트하려고 `FlutterLocalNotificationsPlugin`을 직접 호출하면:

```
MissingPluginException(No implementation found for method initialize
on channel dexterous.com/flutter/local_notifications)
```

CI 환경에는 네이티브 플러그인 바인딩 자체가 없으니 당연한 얘기다. `setUpAll`에서 `TestWidgetsFlutterBinding.ensureInitialized()`를 붙여봤자 해결 안 된다.

처음엔 `MethodChannel`을 직접 mock하는 방법도 써봤는데, `flutter_local_notifications` 내부 채널 구조가 버전마다 달라서 금방 깨졌다. 결국 제대로 된 방법은 하나다 — 추상화.

## NotificationService 인터페이스 정의

추상 클래스 하나로 알림 기능을 감싼다.

```dart
abstract class NotificationService {
  Future<void> initialize();
  Future<void> show({
    required int id,
    required String title,
    required String body,
    String? payload,
  });
  Future<void> schedule({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledAt,
    String? payload,
  });
  Future<void> cancel(int id);
  Future<void> cancelAll();
}
```

실제 구현체는 이 인터페이스를 구현한다.

```dart
class LocalNotificationService implements NotificationService {
  final FlutterLocalNotificationsPlugin _plugin;

  LocalNotificationService(this._plugin);

  @override
  Future<void> initialize() async {
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: true,
      requestBadgePermission: true,
      requestSoundPermission: true,
    );
    await _plugin.initialize(
      const InitializationSettings(android: androidSettings, iOS: iosSettings),
    );
  }

  @override
  Future<void> show({
    required int id,
    required String title,
    required String body,
    String? payload,
  }) async {
    const details = NotificationDetails(
      android: AndroidNotificationDetails(
        'iot_alerts',
        'IoT 기기 알림',
        importance: Importance.high,
        priority: Priority.high,
      ),
      iOS: DarwinNotificationDetails(),
    );
    await _plugin.show(id, title, body, details, payload: payload);
  }

  @override
  Future<void> schedule({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledAt,
    String? payload,
  }) async {
    const details = NotificationDetails(
      android: AndroidNotificationDetails('iot_schedule', 'IoT 예약 알림'),
      iOS: DarwinNotificationDetails(),
    );
    await _plugin.zonedSchedule(
      id,
      title,
      body,
      tz.TZDateTime.from(scheduledAt, tz.local),
      details,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      payload: payload,
    );
  }

  @override
  Future<void> cancel(int id) => _plugin.cancel(id);

  @override
  Future<void> cancelAll() => _plugin.cancelAll();
}
```

## FakeNotificationService 구현

테스트에서 쓸 Fake다. 네이티브를 전혀 건드리지 않고, 호출 내역을 메모리에 쌓아둔다.

```dart
class FakeNotificationService implements NotificationService {
  final List<Map<String, dynamic>> shown = [];
  final List<Map<String, dynamic>> scheduled = [];
  final List<int> cancelled = [];
  bool cancelledAll = false;
  bool initialized = false;

  @override
  Future<void> initialize() async {
    initialized = true;
  }

  @override
  Future<void> show({
    required int id,
    required String title,
    required String body,
    String? payload,
  }) async {
    shown.add({'id': id, 'title': title, 'body': body, 'payload': payload});
  }

  @override
  Future<void> schedule({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledAt,
    String? payload,
  }) async {
    scheduled.add({
      'id': id,
      'title': title,
      'body': body,
      'scheduledAt': scheduledAt,
      'payload': payload,
    });
  }

  @override
  Future<void> cancel(int id) async {
    cancelled.add(id);
  }

  @override
  Future<void> cancelAll() async {
    cancelledAll = true;
  }

  void reset() {
    shown.clear();
    scheduled.clear();
    cancelled.clear();
    cancelledAll = false;
    initialized = false;
  }
}
```

## 알림 로직 Controller 테스트

![Flutter 단위 테스트 코드 화면](https://images.unsplash.com/photo-1555949963-ff9fe0c870eb?w=800&q=80)

IoT 앱에서 실제로 쓰이는 알림 Controller 예시.

```dart
class DeviceAlertController extends GetxController {
  final NotificationService _notification;
  final DeviceRepository _device;

  DeviceAlertController(this._notification, this._device);

  Future<void> checkAndAlert(String deviceId) async {
    final device = await _device.getDevice(deviceId);
    if (device == null) return;

    if (!device.isConnected) {
      await _notification.show(
        id: device.id.hashCode,
        title: '기기 연결 끊김',
        body: '${device.name}과의 연결이 끊어졌습니다.',
        payload: deviceId,
      );
      return;
    }

    if (device.temperature != null && device.temperature! > 80.0) {
      await _notification.show(
        id: device.id.hashCode + 1,
        title: '온도 경고',
        body: '${device.name} 온도가 ${device.temperature}°C입니다.',
        payload: deviceId,
      );
    }
  }

  Future<void> scheduleBoilerAlert(String deviceId, DateTime at) async {
    await _notification.schedule(
      id: deviceId.hashCode,
      title: '보일러 예약',
      body: '예약 시간이 됐습니다.',
      scheduledAt: at,
      payload: deviceId,
    );
  }

  Future<void> cancelDeviceAlerts(String deviceId) async {
    await _notification.cancel(deviceId.hashCode);
    await _notification.cancel(deviceId.hashCode + 1);
  }
}
```

이제 이걸 테스트한다. 네이티브 없음, 실기기 없음.

```dart
void main() {
  late FakeNotificationService fakeNotification;
  late FakeDeviceRepository fakeDevice;
  late DeviceAlertController controller;

  setUp(() {
    fakeNotification = FakeNotificationService();
    fakeDevice = FakeDeviceRepository();
    controller = DeviceAlertController(fakeNotification, fakeDevice);
  });

  tearDown(() {
    fakeNotification.reset();
    Get.reset();
  });

  group('DeviceAlertController', () {
    test('연결 끊긴 기기 — 알림 1개 발송', () async {
      fakeDevice.add(Device(
        id: 'boiler-01',
        name: '거실 보일러',
        isConnected: false,
      ));

      await controller.checkAndAlert('boiler-01');

      expect(fakeNotification.shown.length, 1);
      expect(fakeNotification.shown.first['title'], '기기 연결 끊김');
      expect(fakeNotification.shown.first['payload'], 'boiler-01');
    });

    test('온도 80도 초과 — 경고 알림 발송', () async {
      fakeDevice.add(Device(
        id: 'sensor-01',
        name: '온도 센서',
        isConnected: true,
        temperature: 85.0,
      ));

      await controller.checkAndAlert('sensor-01');

      expect(fakeNotification.shown.length, 1);
      expect(fakeNotification.shown.first['title'], '온도 경고');
      expect(fakeNotification.shown.first['body'], contains('85.0'));
    });

    test('정상 기기 — 알림 없음', () async {
      fakeDevice.add(Device(
        id: 'light-01',
        name: '조명',
        isConnected: true,
        temperature: 35.0,
      ));

      await controller.checkAndAlert('light-01');

      expect(fakeNotification.shown, isEmpty);
    });

    test('예약 알림 — schedule 호출 확인', () async {
      final scheduledAt = DateTime(2026, 6, 29, 7, 0);
      await controller.scheduleBoilerAlert('boiler-01', scheduledAt);

      expect(fakeNotification.scheduled.length, 1);
      expect(fakeNotification.scheduled.first['scheduledAt'], scheduledAt);
      expect(fakeNotification.scheduled.first['title'], '보일러 예약');
    });

    test('알림 취소 — 기기 ID 기반 cancel 2회 호출', () async {
      await controller.cancelDeviceAlerts('boiler-01');

      expect(fakeNotification.cancelled.length, 2);
      expect(fakeNotification.cancelled, contains('boiler-01'.hashCode));
    });
  });
}
```

## 삽질한 부분

**`zonedSchedule` timezone 초기화 누락**

실제 구현에서 `schedule`을 처음 구현할 때 앱 시작 시 timezone 초기화를 빠뜨렸다.

```dart
// main.dart에 이게 없으면 zonedSchedule이 런타임에서 터진다
import 'package:timezone/data/latest.dart' as tz;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  tz.initializeTimeZones(); // 이게 없으면 scheduledAt 변환에서 예외
  runApp(const MyApp());
}
```

Fake에서는 timezone을 안 쓰니까 테스트는 통과한다. 근데 실기기에서 예약 알림을 눌러보기 전까지는 이 버그를 모른다. 통합 테스트까지 도달해서야 발견했다.

**notification ID 충돌**

처음에는 그냥 `Random().nextInt(999999)`로 ID를 뽑았다. 테스트에서 ID를 검증하려니 매번 달라서 assertion이 의미 없었다. `deviceId.hashCode`로 고정하면 같은 기기의 알림은 항상 같은 ID라 재발송 시 자동으로 덮어쓰이고, 취소도 예측 가능하다.

**초기화 순서 — initialize() 먼저**

Fake에서는 `initialize()` 안 해도 `show()`가 잘 된다. 실제 구현은 `initialize()` 전에 `show()` 하면 아무 일도 일어나지 않는다(에러도 안 난다). 이 차이를 모르고 초기화 로직을 빠뜨린 채 배포했다가 실기기에서 알림이 하나도 안 뜨는 상황을 겪었다.

Fake에 `initialized` 플래그를 넣은 이유가 여기 있다. `initialize()` 없이 `show()` 하면 예외를 던지도록 strict하게 만들어두면 더 좋다.

```dart
@override
Future<void> show({...}) async {
  if (!initialized) throw StateError('initialize()를 먼저 호출해야 합니다.');
  shown.add({...});
}
```

---

다음 편은 `easy_localization`이 포함된 위젯에서 다국어 텍스트를 단위 테스트로 검증하는 방법이다. `TranslationAssets` Fake를 어떻게 주입하는지, CI에서 assets 로딩 이슈를 어떻게 피하는지 다룬다.

**참고 링크**
- [flutter_local_notifications pub.dev](https://pub.dev/packages/flutter_local_notifications)
- [timezone 패키지](https://pub.dev/packages/timezone)
- [{% post_url 2026-06-26-14-00-00-777735-flutter-fake-async-stream-timer-unit-test %}]({% post_url 2026-06-26-14-00-00-777735-flutter-fake-async-stream-timer-unit-test %}) — fakeAsync로 타이머·스트림 테스트
- [{% post_url 2026-06-26-20-00-00-814770-flutter-firebase-test-isolation-auth-remote-config-fake %}]({% post_url 2026-06-26-20-00-00-814770-flutter-firebase-test-isolation-auth-remote-config-fake %}) — Firebase 격리 패턴
