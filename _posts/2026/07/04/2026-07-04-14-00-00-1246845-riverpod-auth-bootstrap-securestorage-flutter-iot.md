---
layout: post
title: "Flutter IoT Riverpod auth bootstrap - SecureStorage 토큰 복원 중 화면 튐 막기"
description: "Flutter IoT 앱에서 Riverpod 인증 Provider와 flutter_secure_storage로 JWT 세션을 복원할 때 Flutter 스마트홈 첫 화면이 로그인으로 튀는 문제를 정리했다."
date: 2026-07-04
tags: [Flutter, Riverpod, flutter_secure_storage, JWT, 인증, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod auth bootstrap](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 앱 시작 직후 인증 저장소, 라우터, IoT 세션이 같은 타이밍에 움직이는 지점을 보면 된다.

Flutter IoT 앱에서 Riverpod 인증 Provider를 만들 때 `flutter_secure_storage`에서 JWT를 읽는 200~500ms를 별도 상태로 잡아야 한다. Flutter 스마트홈 앱은 이 짧은 구간을 놓치면 로그인 화면을 잠깐 보여준 뒤 홈으로 이동하거나, MQTT 세션을 두 번 초기화한다.

처음엔 `token == null`이면 미로그인, 값이 있으면 로그인으로만 나눴다. 해보니 앱 실행 직후가 문제였다. SecureStorage 읽기가 끝나기 전에는 토큰이 없는 게 아니라 "아직 모르는 상태"다. 이 상태를 미로그인으로 처리하니 GoRouter가 `/login`으로 보냈고, 0.3초 뒤 토큰이 복원되면서 `/home`으로 다시 이동했다. 화면이 한 번 튀는 정도면 참고 넘어갈 수 있는데, MQTT 연결까지 붙어 있으면 로그가 지저분해졌다.

| 상태 | 의미 | 라우팅 |
|---|---|---|
| `booting` | SecureStorage에서 토큰 확인 중 | 스플래시 유지 |
| `authenticated` | access token 또는 refresh token 유효 | 홈 진입 |
| `expired` | 토큰은 있으나 갱신 실패 | 로그인 이동 |
| `signedOut` | 저장된 토큰 없음 | 로그인 이동 |

핵심은 `AuthState`에 부팅 상태를 넣는 것이다. 라우터가 저장소를 직접 읽지 않고, Provider가 초기화를 끝낸 결과만 보게 만든다.

```dart
sealed class AuthState {
  const AuthState();
}

class AuthBooting extends AuthState {
  const AuthBooting();
}

class Authenticated extends AuthState {
  const Authenticated(this.userId);
  final String userId;
}

class SignedOut extends AuthState {
  const SignedOut();
}

final authNotifierProvider =
    AsyncNotifierProvider<AuthNotifier, AuthState>(AuthNotifier.new);

class AuthNotifier extends AsyncNotifier<AuthState> {
  @override
  Future<AuthState> build() async {
    final tokenRepository = ref.read(tokenRepositoryProvider);
    final token = await tokenRepository.readToken();

    if (token == null) {
      return const SignedOut();
    }

    final userId = await tokenRepository.restoreSession(token);
    return Authenticated(userId);
  }
}
```

여기서 헷갈렸던 건 `AsyncLoading`만으로 충분하다고 생각한 부분이다. 단순 화면이면 그럴 수 있다. 그런데 IoT 앱은 인증 복원 뒤에 Space 목록 조회, MQTT subscribe, BLE 자동 재연결 후보 확인이 이어진다. `loading`을 화면 로딩으로만 보면 통신 세션 초기화와 구분이 안 된다. 나는 인증 부팅 중에는 라우터만 잠그고, 대시보드 데이터 로딩은 홈 화면 안에서 따로 처리하는 쪽이 덜 꼬였다.

라우터 쪽은 판단만 한다.

```dart
redirect: (context, state) {
  final auth = container.read(authNotifierProvider);

  if (auth.isLoading) {
    return '/splash';
  }

  final value = auth.valueOrNull;
  if (value is Authenticated) {
    return state.matchedLocation == '/login' ? '/home' : null;
  }

  return state.matchedLocation == '/login' ? null : '/login';
}
```

주의할 점은 토큰 복원 실패를 조용히 삼키지 않는 것이다. `restoreSession()`에서 refresh token 갱신이 실패했다면 SecureStorage를 지우고 `SignedOut`으로 내려야 한다. 실패한 토큰을 남겨두면 앱을 켤 때마다 `booting -> expired -> login`이 반복된다. 실제로 iOS 실기기에서 키체인 값은 남아 있는데 서버 세션만 만료된 케이스가 있었다. 이때 MQTT는 연결을 시도하지 않아야 하고, Crashlytics에는 토큰 원문이 아니라 실패 코드만 남겨야 한다.

짧게 남기면 이렇다. Flutter IoT 앱의 인증 Provider는 로그인 여부만 들고 있으면 부족하다. SecureStorage 확인 중인 부팅 상태, 토큰 갱신 실패, 로그아웃을 분리해야 GoRouter 화면 튐과 MQTT 중복 초기화를 막을 수 있다. 인증이 안정돼야 Flutter BLE 재연결과 스마트홈 대시보드 상태도 덜 흔들린다.
