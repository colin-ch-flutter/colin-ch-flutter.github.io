---
layout: post
title: "Flutter integration_test JWT 자동 갱신 - 401 동시 요청 경쟁 조건 검증"
description: "Flutter integration_test에서 JWT가 만료된 순간 MQTT·기기 API 요청이 동시에 401을 받을 때 refresh token을 한 번만 사용하고 원 요청을 안전하게 재실행하는 방법을 정리했다."
date: 2026-08-30
tags: [Flutter, Dart, integration_test, JWT, SecureStorage, IoT, 스마트홈]
comments: true
share: true
---

![Flutter integration_test JWT 자동 갱신과 IoT 요청 재실행](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Flutter integration_test에서 로그인 후 보일러 상태를 읽는 흐름만 통과시키면 JWT 만료 버그를 놓치기 쉽다. 실제 스마트홈 앱에서는 기기 목록, MQTT 세션, 알림 동기화 요청이 거의 동시에 나간다. 토큰이 만료된 순간 세 요청이 모두 `401`을 받으면 refresh token을 세 번 쓰거나, 한 요청만 재실행되고 나머지는 로그아웃되는 문제가 생긴다. 결론은 **동시 401을 하나의 refresh 작업으로 합치고, 각 원 요청의 재실행까지 확인하는 것**이다.

## 만료 토큰은 단독 요청보다 동시에 터진다

처음에는 API 클라이언트에서 `401`이면 곧바로 `refresh()`를 호출했다. 단일 테스트에서는 잘 됐다. 하지만 앱을 콜드 스타트하면 `GET /devices`, `GET /spaces`, MQTT 인증이 함께 시작된다. Fake 서버가 세 응답을 연달아 `401`로 반환하자 refresh API가 세 번 호출됐고, 서버는 두 번째 요청부터 이미 사용한 refresh token으로 판단했다.

| 상황 | 잘못된 결과 | 검증해야 할 기준 |
|---|---|---|
| 401 한 번 | 요청만 실패 | refresh 후 원 요청 1회 재실행 |
| 401 세 번 동시 발생 | refresh 3회, 세션 삭제 | refresh 1회, 세 요청 모두 성공 |
| refresh 실패 | 무한 재시도 | 저장소 삭제 후 로그인 화면 이동 |

핵심은 `Future`를 공유하는 것이다. 이미 갱신 중이면 새 refresh를 시작하지 않고 기존 Future를 기다린다. 아래 코드는 실제 프로젝트의 `AuthInterceptor`를 단순화한 형태다.

```dart
class TokenRefresher {
  TokenRefresher(this.authApi, this.storage);

  final AuthApi authApi;
  final TokenStorage storage;
  Future<String>? _refreshing;

  Future<String> refreshOnce() {
    final running = _refreshing;
    if (running != null) return running;

    final future = _refreshToken();
    _refreshing = future;
    future.whenComplete(() => _refreshing = null);
    return future;
  }

  Future<String> _refreshToken() async {
    final refreshToken = await storage.readRefreshToken();
    if (refreshToken == null) throw const AuthExpired();

    final token = await authApi.refresh(refreshToken);
    await storage.saveAccessToken(token.accessToken);
    await storage.saveRefreshToken(token.refreshToken);
    return token.accessToken;
  }
}
```

## integration_test에서는 만료 순서를 직접 만든다

실제 인증 서버의 만료 시각을 기다리면 테스트가 느리고 불안정하다. `FakeAuthApi`에 `401` 응답 횟수와 refresh 호출 횟수를 넣고, 앱 시작 직후 세 요청을 동시에 발생시켰다.

```dart
testWidgets('동시 401은 refresh 한 번 후 모든 요청을 재실행한다',
    (tester) async {
  final auth = FakeAuthApi(
    unauthorizedCalls: 3,
    refreshedAccessToken: 'access-2',
  );
  await launchTestApp(authApi: auth);

  await tester.tap(find.byKey(const Key('open-dashboard')));
  await tester.pumpAndSettle();

  expect(auth.refreshCalls, 1);
  expect(auth.retriedPaths, containsAll(<String>[
    '/spaces',
    '/devices',
    '/mqtt/session',
  ]));
  expect(find.text('거실 보일러'), findsOneWidget);
  expect(find.text('로그인 만료'), findsNothing);
});
```

여기서 `pumpAndSettle()`만 호출하는 것은 충분하지 않다. Fake의 refresh Future를 테스트 안에서 완료시키지 않으면 앱이 계속 대기할 수 있다. 실제 헬퍼에서는 `Completer<Token>`을 외부에 노출해 `refreshStarted`를 확인한 뒤 완료시킨다. 또 재실행 요청에 이미 새 access token이 들어갔는지와 같은 요청이 두 번 이상 실행되지 않았는지를 별도로 기록했다.

## refresh 실패도 성공 시나리오만큼 중요하다

refresh token이 만료된 경우 interceptor가 다시 `401`을 만들면 무한 루프가 시작된다. 재시도 횟수는 원 요청당 1회로 제한하고, 갱신 실패 시 `flutter_secure_storage`의 access·refresh token을 함께 지운 뒤 로그인 화면으로 보내야 한다. 테스트에서는 `refreshThrows = true`로 바꾸고, `/devices`가 두 번 이상 호출되지 않는지 확인했다.

주의할 점은 이 테스트가 서버의 JWT 서명 검증이나 실제 MQTT 브로커 세션까지 보장하지 않는다는 것이다. 목적은 토큰 만료 시 앱 내부의 동시성 계약을 고정하는 데 있다. 실기기에서는 앱 시작 시점, 백그라운드 복귀 시점에 각각 한 번씩 실제 refresh를 확인하고, CI에서는 Fake로 모든 요청의 순서를 반복하는 구성이 현실적이었다.

짧게 정리하면 `401 감지 → refresh Future 공유 → access token 저장 → 원 요청 1회 재실행`의 순서를 테스트에 남겨야 한다. refresh 호출 횟수만 1인지 보는 것보다, 동시에 실패한 IoT 요청 세 개가 모두 올바른 토큰으로 끝났는지까지 확인해야 세션 경쟁 조건을 잡을 수 있다.
