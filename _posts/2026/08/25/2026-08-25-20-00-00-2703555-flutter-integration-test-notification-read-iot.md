---
layout: post
title: "Flutter integration_test IoT 알림 읽음 처리 - FCM 배지 상태 검증"
description: "Flutter integration_test에서 IoT 알림 수신부터 읽음 처리까지 검증하고, FCM 이벤트와 Realm 배지 상태가 어긋나는 문제를 막는 실전 패턴을 정리했다."
date: 2026-08-25
tags: [Flutter, Dart, integration_test, Firebase, FCM, Realm, IoT, 스마트홈]
comments: true
share: true
---

![Flutter integration_test IoT 알림 읽음 처리와 배지 상태 검증](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Flutter integration_test에서 IoT 알림이 도착했다는 것만 확인하면 부족하다. 알림센터에 카드가 생기고, 읽음 버튼을 누른 뒤 앱 배지가 0으로 바뀌며, 앱을 다시 열어도 읽음 상태가 유지되는 흐름까지 검증해야 한다. 처음엔 FCM 실서버에서 메시지를 쏘면 된다고 생각했는데, 해보니 토큰 만료와 네트워크 타이밍 때문에 테스트가 자주 흔들렸다. 결론은 **FCM 전달 자체와 앱 내부 읽음 상태를 분리하고, integration_test에서는 수신 이벤트를 주입하는 것**이다.

## 실제로 깨졌던 지점

보일러가 오프라인이 되면 FCM으로 알림을 보낸다. 앱이 포그라운드면 인앱 이벤트를 받고, 백그라운드면 시스템 알림을 누른 뒤 알림센터로 들어온다. 두 경로가 결국 같은 `NotificationRepository`를 거쳐야 하는데, 한쪽은 메모리 배지만 줄이고 Realm에는 저장하지 않는 버그가 있었다.

| 검증 단계 | 확인할 값 | 놓치기 쉬운 실패 |
| --- | --- | --- |
| 이벤트 수신 | 알림 ID와 기기 ID | 같은 알림이 두 번 추가됨 |
| 센터 진입 | 읽지 않은 카드 1개 | 배지만 있고 카드가 없음 |
| 읽음 처리 | `isRead == true` | 화면만 바뀌고 DB는 미변경 |
| 재시작 | 배지 0, 읽음 유지 | 콜드 스타트에서 다시 unread |

## 수신 이벤트를 테스트에서 주입하기

FCM SDK를 직접 기다리지 않고 앱이 실제로 받는 payload 형태만 주입한다. 중요한 건 테스트용 Fake도 운영 코드와 동일한 인터페이스를 사용하게 만드는 것이다.

```dart
class FakeNotificationGateway implements NotificationGateway {
  final _controller = StreamController<IoTNotification>.broadcast();

  @override
  Stream<IoTNotification> get events => _controller.stream;

  void emit({required String id, required String deviceId}) {
    _controller.add(IoTNotification(
      id: id,
      deviceId: deviceId,
      title: '보일러 연결 끊김',
      isRead: false,
    ));
  }

  @override
  Future<void> dispose() => _controller.close();
}
```

이 Fake는 FCM을 흉내 내는 목적이 아니다. `NotificationController`가 이벤트를 받아 Repository에 저장하고, Stream을 구독 해제하는지 확인하는 테스트 경계다. 실제 FCM 토큰 발급·OS 알림 표시 테스트는 Firebase Test Lab이나 수동 실기기 시나리오로 따로 둔다.

## 읽음 처리까지 한 시나리오로 검증하기

읽음 버튼을 탭한 직후 화면만 확인하지 않고, Realm 재조회와 앱 재시작까지 넣었다. `pumpAndSettle()`만 믿으면 비동기 저장이 끝나기 전에 통과할 수 있어서, 화면에 표시되는 배지와 저장된 값을 각각 확인했다.

```dart
testWidgets('IoT 알림을 읽으면 배지와 로컬 상태가 함께 갱신된다', (tester) async {
  final gateway = FakeNotificationGateway();
  final repository = FakeNotificationRepository();

  await tester.pumpWidget(TestApp(
    notificationGateway: gateway,
    notificationRepository: repository,
  ));

  gateway.emit(id: 'notice-001', deviceId: 'boiler-001');
  await tester.pump();

  expect(find.text('1'), findsOneWidget); // 미읽음 배지
  await tester.tap(find.byKey(const Key('notification-center')));
  await tester.pumpAndSettle();
  expect(find.text('보일러 연결 끊김'), findsOneWidget);

  await tester.tap(find.byKey(const Key('read-notice-notice-001')));
  await tester.pump();

  expect(find.text('0'), findsOneWidget);
  expect((await repository.findById('notice-001')).isRead, isTrue);
});
```

여기서 테스트가 처음 실패한 이유는 `tap()` 뒤에 Repository 저장을 기다리는 상태가 없었기 때문이다. 읽음 처리 메서드가 `Future`를 반환하도록 바꾸고, Controller에서 저장 완료 후 상태를 갱신하니 재현되던 경합이 사라졌다. 알림을 두 번 수신했을 때 ID 기준 upsert가 되는지도 별도 assertion으로 넣었다.

## 앱 재시작 뒤에도 배지가 살아 있는지

integration_test에서는 첫 실행만 보면 안 된다. 테스트 데이터를 Realm에 넣은 뒤 앱을 재실행하고, `NotificationController.onReady()`가 unread 개수를 다시 계산하는지 확인한다.

```dart
await driver.restart();
await tester.pumpAndSettle();

expect(find.byKey(const Key('unread-count-0')), findsOneWidget);
expect(find.text('보일러 연결 끊김'), findsOneWidget);
```

운영 환경의 FCM 도착 여부와 앱 상태 복원은 서로 다른 책임이다. 전자는 전달 모니터링으로, 후자는 이 테스트로 잡는 편이 실패 원인을 훨씬 빨리 좁힐 수 있다.

짧게 정리하면, `FCM 수신 → Repository 저장 → 알림센터 표시 → 읽음 저장 → 재시작 복원`을 한 흐름으로 묶어야 한다. 특히 배지를 먼저 0으로 만들고 저장하는 구현은 앱이 죽는 순간 상태가 되돌아갈 수 있으니, 저장 완료 뒤 UI 상태를 바꾸는 순서가 안전하다.
