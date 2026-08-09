---
layout: post
title: "Flutter integration_test FCM 알림 테스트 - IoT 경보 화면까지 검증하기"
description: "Flutter integration_test에서 Firebase FCM 알림을 받은 뒤 스마트홈 경보 화면으로 이동하는 흐름을 테스트하는 방법과 실기기에서 놓치기 쉬운 앱 상태별 차이를 정리했다."
date: 2026-08-10
tags: [Flutter, Dart, Firebase, FCM, IoT, Android, iOS, CI/CD]
comments: true
share: true
---

![Flutter integration_test와 스마트홈 FCM 알림 테스트](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

이 그림에서 봐야 할 부분은 알림을 받는 순간이 아니라, 사용자가 알림을 눌렀을 때 앱이 올바른 IoT 경보 화면과 기기로 이동하는지다.

Flutter integration_test에서 버튼을 눌러 MQTT 명령을 보내는 흐름은 이미 익숙하다. 그런데 실제 스마트홈 앱에서 더 자주 깨지는 건 “보일러 이상 알림을 누른 뒤 해당 기기 상세 화면이 열리는 흐름”이었다. 앱이 foreground인지 background인지, 알림을 눌러 cold start 되었는지에 따라 FCM 콜백 위치가 달라진다. 처음엔 `onMessageOpenedApp` 하나만 기다리면 되는 줄 알았는데 아니었다.

## 앱 상태별로 진입점이 다르다

FCM 메시지를 받았다는 사실과 알림을 눌렀다는 이벤트를 같은 것으로 처리하면 테스트가 간헐적으로 실패한다.

| 앱 상태 | 확인할 이벤트 | 테스트에서 볼 것 |
|---|---|---|
| foreground | `onMessage` | 앱 안에 경보 배너 표시 |
| background | `onMessageOpenedApp` | 알림 탭 후 기기 상세 이동 |
| terminated | `getInitialMessage()` | cold start 뒤 초기 라우팅 |

세 경우 모두 payload의 `deviceId`가 필요하다. 제목 문자열을 보고 화면을 찾게 만들면 다국어 설정이나 알림 표시 형식이 바뀌는 순간 테스트가 흔들린다. 라우팅에 필요한 값은 데이터 payload에 고정했다.

## 알림 처리를 서비스로 감싸기

Firebase 인스턴스를 테스트 코드가 직접 만지지 않도록, 앱에서 사용하는 알림 서비스에 이벤트를 흘려보내게 했다. 아래 코드는 실제 서비스에서 라우터로 전달할 이벤트를 단순화한 형태다.

```dart
class NotificationIntent {
  const NotificationIntent(this.deviceId, this.type);

  final String deviceId;
  final String type;
}

class FcmService {
  FcmService(this._messaging, this._intents);

  final FirebaseMessaging _messaging;
  final StreamController<NotificationIntent> _intents;

  Future<void> initialize() async {
    FirebaseMessaging.onMessageOpenedApp.listen(_emit);
    final initial = await _messaging.getInitialMessage();
    if (initial != null) _emit(initial);
  }

  void _emit(RemoteMessage message) {
    final data = message.data;
    final deviceId = data['deviceId'];
    if (deviceId is String && data['type'] == 'device_alert') {
      _intents.add(NotificationIntent(deviceId, data['type'] as String));
    }
  }
}
```

핵심은 `initialize()`를 `runApp()` 이후 한 번만 호출하고, 라우터가 준비된 뒤 intent를 소비하는 것이다. 초기화보다 알림 이벤트가 먼저 도착하면 화면 이동이 유실될 수 있어서, 실제 앱에서는 `PendingIntentStore`에 잠시 보관했다가 첫 화면이 준비된 뒤 읽었다.

## integration_test에서는 FCM 서버를 직접 부르지 않는다

실기기 테스트마다 Firebase 서버에서 실제 푸시를 보내면 토큰 만료, 전송 지연, 기기별 등록 상태가 테스트 결과에 섞인다. 앱에 테스트 전용 `NotificationIntent` 주입구를 만들고, integration_test에서는 같은 사용자 동작을 흉내 냈다.

```dart
testWidgets('보일러 경보 알림을 누르면 해당 기기로 이동한다', (tester) async {
  await app.main(testMode: true);
  await tester.pumpAndSettle();

  await TestNotificationGateway.emit({
    'type': 'device_alert',
    'deviceId': 'boiler-living-01',
  });
  await tester.pump(const Duration(milliseconds: 300));

  expect(find.byKey(const Key('device-detail-boiler-living-01')),
      findsOneWidget);
  expect(find.text('보일러 이상'), findsOneWidget);
});
```

여기서 `deviceId`에 공백이 들어간 예시는 일부러 실패했던 fixture를 단순화한 것이다. 서버 payload에는 사람이 읽는 이름이 아니라 정규화된 ID만 넣어야 한다. 나는 처음에 `living room boiler`를 그대로 보냈다가 URL 인코딩과 키 비교에서 테스트가 깨졌다. fixture 생성 단계에서 정규식으로 허용 문자를 검사하고, 실패하면 payload 전체를 로그에 남기도록 고쳤다.

## 실기기에서 확인할 기준

Android는 앱을 백그라운드로 보낸 뒤 알림을 탭하는 흐름을 비교적 쉽게 자동화할 수 있지만, iOS는 알림 권한과 앱 종료 상태를 테스트 준비 단계에서 맞춰야 한다. 테스트가 시작될 때 권한 팝업이 떠 있으면 `pumpAndSettle()`은 화면이 안정될 때까지 기다릴 뿐 팝업을 눌러주지 않는다.

체크 기준은 이 정도로 충분했다.

- foreground에서는 배너가 중복으로 두 번 생기지 않는가
- background에서는 알림 탭 뒤 `deviceId`가 유지되는가
- terminated에서는 로그인 복원 뒤 pending intent가 소비되는가
- 잘못된 `type`이나 없는 `deviceId`가 홈 화면을 깨뜨리지 않는가

정리하면 FCM 테스트의 핵심은 실제 푸시 전송 성공 여부가 아니라, 앱 상태별 이벤트를 같은 `NotificationIntent`로 수렴시키는 데 있다. Firebase를 외부 시스템으로 남겨두고 intent를 주입할 수 있게 만들면 Flutter integration_test도 IoT 경보와 딥링크를 재현 가능하게 검증할 수 있다.
