---
layout: post
title: "Flutter integration_test 앱 시작 경쟁 조건 - Splash와 AuthGuard 검증"
description: "Flutter integration_test에서 Splash 초기화와 AuthGuard가 경쟁해 잘못된 로그인 화면이 보이는 문제를 Fake 세션과 pumpUntilVisible로 검증하는 방법을 정리했다."
date: 2026-08-27
tags: [Flutter, Dart, IoT, 스마트홈, GetX, CI/CD]
comments: true
share: true
---

![Flutter integration_test 앱 시작과 AuthGuard 검증](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter integration_test에서 Splash가 사라지고 스마트홈 화면이 열리는지만 확인하면 부족하다. 세션 복원이 끝나기 전에 `AuthGuard`가 판단하면 로그인 화면이 먼저 나타났다가 다시 IoT 화면으로 튀는 경쟁 조건을 놓치기 쉽다. 처음엔 `pumpAndSettle()` 한 번이면 될 줄 알았는데, 해보니 CI에서만 실패했다.

## 문제는 Splash가 아니라 초기화 순서였다

내 앱의 시작 흐름은 아래와 같았다.

| 단계 | 정상 흐름 | 경쟁 조건이 생길 때 |
|---|---|---|
| 앱 실행 | Splash 표시 | Splash 표시 |
| SecureStorage 조회 | 세션 복원 대기 | AuthGuard가 빈 토큰을 읽음 |
| 라우팅 | Space 화면 진입 | 로그인 화면으로 이동 |
| 복원 완료 | 동일 화면 유지 | 뒤늦게 IoT 화면으로 재이동 |

`null`을 곧바로 로그아웃으로 해석하면 느린 기기에서 실패 확률이 올라간다.

## 세션 상태를 세 가지로 나눈다

`AuthGuard`가 초기화 중인 세션을 로그인 실패로 오해하지 않도록 상태를 분리한다.

```dart
enum SessionStatus { loading, authenticated, unauthenticated }

class FakeSessionRepository implements SessionRepository {
  FakeSessionRepository({this.status = SessionStatus.loading});

  SessionStatus status;
  final completer = Completer<void>();

  @override
  Future<SessionStatus> restore() async {
    await completer.future;
    return status;
  }

  void complete({required SessionStatus next}) {
    status = next;
    completer.complete();
  }
}
```

실제 저장소 대신 복원 완료 시점을 제어하는 Fake를 주입한다.

## 테스트에서 세션 복원보다 라우팅이 먼저 실행되는지 확인한다

앱을 띄운 직후 로그인 화면을 찾지 않고, 복원 전에는 Splash가 유지되는지 확인한다.

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('세션 복원 전에는 AuthGuard가 로그인으로 보내지 않는다',
      (tester) async {
    final session = FakeSessionRepository();

    await tester.pumpWidget(
      SmartHomeApp(sessionRepository: session),
    );
    await tester.pump();

    expect(find.byKey(const Key('splash')), findsOneWidget);
    expect(find.byKey(const Key('login-screen')), findsNothing);

    session.complete(next: SessionStatus.authenticated);
    await tester.pumpAndSettle();

    expect(find.byKey(const Key('space-screen')), findsOneWidget);
    expect(find.byKey(const Key('login-screen')), findsNothing);
  });
}
```

시작 직후 `pumpAndSettle()`을 호출하면 경쟁 조건을 숨길 수 있다. Fake 세션을 완료하기 전에는 `pump()`만 호출한다.

## 실제로 잡힌 실패와 주의사항

처음에는 `status != authenticated`일 때 무조건 `/login`으로 보냈다. Android 에뮬레이터에서만 로그인 화면이 간헐적으로 캡처됐다. `loading`일 때 Splash를 반환하도록 고친 뒤 실패가 사라졌다.

딥링크가 함께 들어오면 `pendingRoute`를 세션 복원 완료까지 보관한다. 인증된 링크는 Space 목록을 기다린 뒤 한 번만 소비해야 MQTT 구독 전 화면 진입을 막을 수 있다.

검증 기준은 복원 전 `Splash` 유지, `loading` 상태에서 로그인 라우트 미호출, 복원 후 Space 화면 진입 1회다. Flutter integration_test에서는 초기화 상태와 완료 시점을 주입해야 CI에서도 이 경쟁 조건을 재현할 수 있다.
