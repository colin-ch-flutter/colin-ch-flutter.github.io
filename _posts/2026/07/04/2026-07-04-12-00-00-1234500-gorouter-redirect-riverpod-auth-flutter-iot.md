---
layout: post
title: "Flutter IoT GoRouter redirect - Riverpod 인증 상태로 GetX 라우팅 걷어내기"
description: "Flutter IoT 앱에서 GoRouter redirect와 Riverpod 인증 Provider로 Flutter 스마트홈 로그인 리다이렉트와 MQTT 세션 초기화를 분리한 기준을 정리했다."
date: 2026-07-04
tags: [Flutter, Riverpod, GetX, 인증, IoT, 스마트홈]
comments: true
share: true
---
![Flutter IoT GoRouter redirect와 Riverpod 인증 상태](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 라우터, 인증 상태, IoT 세션이 한 덩어리로 묶이지 않게 분리하는 흐름을 보면 된다.

Flutter IoT 앱에서 GetX 라우팅을 걷어낼 때 핵심은 화면 이동 코드를 한 번에 바꾸는 게 아니었다. Flutter 스마트홈 앱의 로그인 상태, JWT 만료, MQTT 세션 초기화가 `Get.offAllNamed('/login')` 주변에 섞여 있었고, 이걸 그대로 GoRouter로 옮기면 라이브러리만 바뀐 같은 코드가 된다. 내가 확인한 기준으론 Riverpod 인증 Provider가 상태를 들고, GoRouter `redirect`는 이동 판단만 맡는 구조가 가장 덜 흔들렸다.

[GetX 라우팅 테스트 글]({% post_url 2026-06-28-10-00-00-876495-flutter-getx-routing-navigation-unit-test %})에서 라우팅을 단위 테스트로 검증했지만, 실제 마이그레이션은 더 까다로웠다. GetX에서는 Controller 안에서 토큰 삭제, 스낵바, 화면 이동을 한 번에 처리하기 쉬웠다. 편하긴 한데 로그인 만료가 MQTT 연결 끊김과 동시에 오면 순서가 꼬였다. 화면은 로그인으로 갔는데 MQTT 세션 Provider가 살아 있어서 로그에 publish 실패가 계속 남는 식이었다.

| 역할 | GetX에서 흔한 위치 | Riverpod + GoRouter 기준 |
|---|---|---|
| 로그인 여부 | AuthController Rx 값 | `authStateProvider` |
| 이동 판단 | GetMiddleware, Controller | `GoRouter.redirect` |
| 토큰 삭제 | Controller 메서드 내부 | Auth Repository |
| MQTT 정리 | 로그아웃 버튼 근처 | 세션 Provider invalidate |

라우터는 인증 상태를 읽어서 통과 여부만 판단하게 둔다.

```dart
final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authStateProvider);

  return GoRouter(
    initialLocation: '/home',
    redirect: (context, state) {
      final isLoginRoute = state.matchedLocation == '/login';
      final isLoggedIn = authState.valueOrNull?.isLoggedIn == true;

      if (!isLoggedIn && !isLoginRoute) return '/login';
      if (isLoggedIn && isLoginRoute) return '/home';
      return null;
    },
    routes: [
      GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
      GoRoute(path: '/home', builder: (_, __) => const HomePage()),
    ],
  );
});
```

처음엔 `redirect` 안에서 토큰 갱신까지 하려고 했다. 해보니 별로였다. `redirect`는 자주 호출되고, 비동기 갱신이 섞이면 같은 화면에서 두 번 이동하는 현상이 생겼다. 특히 앱 시작 직후 SecureStorage에서 토큰을 읽는 200~500ms 사이에 `/login`을 잠깐 보여줬다가 `/home`으로 튀는 화면이 보였다. 그래서 인증 초기화는 Provider 쪽에서 끝내고, 라우터는 그 결과만 본다.

```dart
final authStateProvider =
    AsyncNotifierProvider<AuthNotifier, AuthState>(AuthNotifier.new);

class AuthNotifier extends AsyncNotifier<AuthState> {
  @override
  Future<AuthState> build() async {
    final repo = ref.read(authRepositoryProvider);
    final session = await repo.restoreSession();
    return AuthState(isLoggedIn: session != null);
  }

  Future<void> logout() async {
    await ref.read(authRepositoryProvider).clearTokens();
    ref.invalidate(mqttSessionProvider);
    ref.invalidate(dashboardProvider);
    state = const AsyncData(AuthState(isLoggedIn: false));
  }
}
```

여기서 헷갈렸던 건 `redirect`가 로그아웃의 주체가 아니라는 점이다. 로그아웃 버튼은 AuthNotifier를 호출하고, AuthNotifier가 토큰과 IoT 세션을 정리한다. 그 결과로 `authStateProvider`가 바뀌면 GoRouter가 `/login`으로 보낸다. 순서를 이렇게 잡으니 테스트도 읽기 쉬워졌다.

```dart
test('로그아웃하면 MQTT 세션을 버리고 로그인 상태가 false가 된다', () async {
  final container = ProviderContainer(overrides: [
    authRepositoryProvider.overrideWithValue(FakeAuthRepository()),
    mqttRepositoryProvider.overrideWithValue(FakeMqttRepository()),
  ]);

  await container.read(authStateProvider.future);
  await container.read(authStateProvider.notifier).logout();

  final auth = container.read(authStateProvider).value!;
  expect(auth.isLoggedIn, false);
});
```

주의할 점은 라우터에서 Provider를 너무 많이 보지 않는 것이다. `mqttConnectionProvider`, `bleScanProvider`, `dashboardProvider`까지 `redirect`에서 읽으면 상태 하나 바뀔 때마다 라우터가 반응한다. Flutter BLE 스캔 결과가 바뀌었는데 로그인 리다이렉트를 다시 계산하는 건 이상하다. 라우터는 인증과 온보딩 정도만 보고, IoT 데이터는 화면 Provider 안에서 처리하는 편이 낫다.

짧게 남기면 이렇다. Flutter IoT에서 GoRouter redirect는 GetX 라우팅을 대체하는 도구지만, AuthController 전체를 옮길 장소는 아니다. Riverpod 인증 Provider가 세션 복구와 로그아웃 정리를 맡고, GoRouter는 이동 판단만 한다. MQTT와 Flutter BLE 세션 정리는 로그아웃 상태 변화에 붙여야 화면 이동과 통신 정리가 서로 꼬이지 않는다.
