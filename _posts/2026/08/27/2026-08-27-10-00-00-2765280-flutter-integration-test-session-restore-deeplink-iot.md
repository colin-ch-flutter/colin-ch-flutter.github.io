---
layout: post
title: "Flutter integration_test 세션 복원 검증 - 딥링크 IoT 화면 진입 경쟁 조건 해결"
description: "Flutter integration_test에서 콜드 스타트 후 JWT 세션 복원 전에 딥링크가 실행되는 문제와 IoT 보호 화면 진입을 안정적으로 검증하는 방법을 정리했다."
date: 2026-08-27
tags: [Flutter, Dart, IoT, 스마트홈, CI/CD]
comments: true
share: true
---

![Flutter integration_test 세션 복원과 IoT 딥링크 진입 검증](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter integration_test에서 딥링크와 JWT 세션 복원을 함께 검증하려면 화면이 보였다는 사실만 확인하면 부족하다. 콜드 스타트 직후 라우터가 세션보다 앞서 실행되면 로그인 화면으로 튕기거나 잘못된 MQTT 토픽을 구독할 수 있다. 세션 복원 완료를 앱의 상태로 만들고 그 전이를 기다려야 한다.

## 실제로 깨진 순서

알림을 눌러 다음 주소로 보일러 제어 화면에 진입하는 흐름을 테스트했다.

```text
myhome://space/space-123/device/boiler-7
```

앱이 종료된 상태에서는 `runApp()` 직후 딥링크 콜백이 들어왔다. 기존 코드는 URL을 파싱하자마자 `router.go()`를 호출했고, `SessionService.restore()`는 별도 초기화 Future에서 실행 중이었다. 테스트 로그는 아래처럼 찍혔다.

```text
[deep-link] open space-123/boiler-7
[router] signedIn=false -> /login
[session] restore completed, signedIn=true
```

실제 IoT 앱에서는 이 순간 공간 선택과 MQTT 구독도 건너뛴다. `find.text('보일러')`만 기다리면 이 타이밍 오류를 놓치기 쉽다.

| 검증 지점 | 정상 순서 | 실패 증상 |
| --- | --- | --- |
| 세션 | 저장 토큰 읽기 → 인증 상태 확정 | 로그인 화면으로 잘못 이동 |
| 공간 | 공간 목록 로드 → 대상 공간 선택 | 기본 공간 토픽 구독 |
| 딥링크 | 대상 확인 후 라우팅 | 화면과 MQTT 상태 불일치 |

## 세션 준비를 하나의 경계로 만들기

핵심은 초기화 Future를 테스트가 추측하지 않게 만드는 것이다. 부트스트랩이 세션과 공간 준비 상태를 노출하고, 딥링크는 그 경계 뒤에서만 라우팅한다.

```dart
enum SessionStatus { loading, signedOut, signedIn }

class AppBootstrap {
  AppBootstrap(this.session, this.spaces);
  final SessionService session;
  final SpaceStore spaces;
  SessionStatus status = SessionStatus.loading;

  Future<void> restore() async {
    await session.restore();
    if (!session.isSignedIn) {
      status = SessionStatus.signedOut;
      return;
    }
    await spaces.ensureLoaded();
    status = SessionStatus.signedIn;
  }
}

Future<void> openLink(DeviceLink link) async {
  await bootstrap.restore();
  if (bootstrap.status != SessionStatus.signedIn) {
    router.go('/login', extra: link.rawUrl);
    return;
  }
  router.go('/space/${link.spaceId}/device/${link.deviceId}');
}
```

실제 앱에서는 `restore()` Future를 캐시해 중복 호출을 막았다. `Future.delayed(const Duration(seconds: 1))`로 버티려 했지만 CI가 느린 날에는 같은 오류가 남았다.

## integration_test에서 단계별로 확인하기

테스트는 전체 화면을 한 번에 검사하지 않고 세션 복원과 최종 라우팅을 각각 확인한다. MQTT 재연결 타이머까지 기다리지 않도록 `pump()`을 쓴다.

```dart
testWidgets('콜드 스타트 딥링크가 세션 복원 후 보일러로 이동한다',
    (tester) async {
  final session = FakeSessionService(restoreResult: true,
      restoreDelay: const Duration(milliseconds: 40));
  final spaces = FakeSpaceStore(spaces: [
    Space(id: 'space-123', name: '우리 집'),
  ]);
  await tester.pumpWidget(TestApp(session: session, spaces: spaces));
  await tester.runAsync(() => appLinks.emit(
      'myhome://space/space-123/device/boiler-7'));

  await tester.pump(const Duration(milliseconds: 20));
  expect(find.byKey(const Key('session-loading')), findsOneWidget);
  expect(find.byKey(const Key('boiler-control')), findsNothing);

  await tester.pump(const Duration(milliseconds: 30));
  await tester.pump();
  expect(find.byKey(const Key('boiler-control')), findsOneWidget);
  expect(spaces.selectedId, 'space-123');
});
```

테스트 빌드에서는 `appLinks`, `SessionService`, `SpaceStore`, MQTT 클라이언트를 주입했다. OS의 실제 딥링크 연결은 Android와 iOS에서 별도 확인하지만, 앱 내부 순서가 뒤집히는 문제는 이 테스트로 잡을 수 있다.

## 정리하며 확인한 기준

- 로딩 중 딥링크는 버리지 말고 원본 URL을 보관한다.
- 인증과 공간 준비 완료를 라우팅 조건으로 둔다.
- 화면 표시와 `selectedSpaceId`, MQTT 토픽을 함께 검증한다.
- 시간 지연값이 아니라 Fake의 완료 시점을 제어한다.

처음에는 라우터 테스트 하나면 충분하다고 생각했는데 아니었다. IoT 앱의 콜드 스타트는 UI 문제가 아니라 상태 복원 순서 문제에 가깝다. `integration_test`에서 그 순서를 숫자로 기다리지 않고 상태로 기다리게 만들면 CI에서도 딥링크 회귀를 안정적으로 잡을 수 있다.
