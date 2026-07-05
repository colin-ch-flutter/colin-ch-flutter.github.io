---
layout: post
title: "Flutter IoT Riverpod ProviderScope - 로그아웃 때 스마트홈 캐시 정리하기"
description: "Flutter IoT 앱에서 Riverpod ProviderScope를 로그인 세션 기준으로 나눠 Flutter 스마트홈, Flutter BLE, MQTT 캐시가 로그아웃 뒤 남는 문제를 정리했다."
date: 2026-07-05
tags: [Flutter, Riverpod, CleanArchitecture, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod ProviderScope 로그아웃 캐시](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 앱 세션과 기기 연결 캐시를 같은 수명으로 묶으면 어디서 상태가 새는지 봐야 한다.

Flutter IoT 앱에서 Riverpod `ProviderScope`는 단순히 루트에 한 번 감싸는 위젯이 아니었다. Flutter 스마트홈 앱처럼 Flutter BLE 스캔 결과, MQTT 연결 상태, JWT 로그인 정보가 같이 움직이면 로그아웃 때 캐시가 남는 순간 문제가 된다. 내가 확인한 기준으론 로그인 세션과 앱 전역 설정을 다른 scope로 나누는 쪽이 덜 헷갈렸다.

처음엔 루트 `ProviderScope` 하나로 충분하다고 봤다. GetX에서 Riverpod으로 옮기는 중이었고, Provider를 전역으로 선언하면 어디서든 읽을 수 있으니 편했다. 문제는 테스트 계정으로 로그아웃한 뒤 다른 계정으로 들어갔을 때 나왔다. 대시보드가 0.5초 정도 이전 집의 보일러 상태를 보여줬고, MQTT subscribe 토픽도 이전 `spaceId`를 한 번 물고 있었다. BLE 쪽은 더 애매했다. 스캔 목록에 이미 등록된 기기가 잠깐 남아서 새 기기 등록 화면에서 잘못된 선택을 할 뻔했다.

내가 나눈 기준은 이렇다.

| Provider 종류 | 수명 기준 | 로그아웃 처리 |
|---|---:|---|
| 앱 테마, locale, RemoteConfig | 앱 실행 전체 | 유지 |
| Auth session, JWT, 사용자 정보 | 로그인 세션 | 폐기 |
| Space, 기기 목록, 대시보드 캐시 | 로그인 세션 | 폐기 |
| MQTT client, BLE scan subscription | 연결 목적별 | 명시적으로 종료 |

핵심은 `ProviderScope`를 계정 세션의 경계로 쓰는 것이다. 아래처럼 `sessionKey`가 바뀌면 세션 아래 Provider들이 새로 만들어진다.

```dart
class AppRoot extends ConsumerWidget {
  const AppRoot({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final session = ref.watch(authSessionProvider);

    return ProviderScope(
      key: ValueKey(session.userId),
      overrides: [
        currentUserProvider.overrideWithValue(session.user),
      ],
      child: const SmartHomeShell(),
    );
  }
}
```

이 코드만으로 모든 정리가 끝나는 건 아니다. `ProviderScope`가 갈리면 Provider 캐시는 버려지지만, Repository 안에서 직접 잡은 MQTT 소켓이나 BLE StreamSubscription은 `ref.onDispose`에서 닫아야 한다. 여기서 한 번 실수했다. Provider는 dispose됐는데 Repository가 내부 타이머를 계속 들고 있어서 로그아웃 뒤에도 재연결 로그가 찍혔다.

```dart
final mqttSessionProvider = Provider.autoDispose<MqttSession>((ref) {
  final session = MqttSession();
  ref.onDispose(session.close);
  return session;
});
```

`ProviderScope`를 너무 잘게 쪼개는 것도 별로였다. Space 탭마다 scope를 만들면 화면 이동 때 캐시가 자주 날아가고, MQTT 구독도 불필요하게 흔들렸다. 반대로 루트 하나에 모든 상태를 넣으면 로그아웃 후 정리가 어렵다. 내 기준은 사용자 계정이 바뀌는 지점 하나만 큰 경계로 두고, Space 전환은 `invalidate`나 family 파라미터로 처리하는 쪽이다.

[JWT 자동갱신]({% post_url 2026-05-20-09-27-33-259245-jwt-token-auto-refresh-session-management %})을 만들 때는 토큰 만료만 생각했는데, Riverpod으로 옮기고 보니 토큰보다 캐시 수명이 더 자주 문제를 만들었다. [클린 아키텍처 구조]({% post_url 2026-05-01-11-14-26-024690-clean-architecture-large-flutter-project-structure %})를 지키려면 화면이 직접 캐시를 지우는 대신 세션 경계에서 한 번에 정리되게 만드는 편이 낫다.

짧게 정리하면 이렇다.

- `ProviderScope`는 앱 루트용 하나만 있는 장식이 아니라 세션 경계를 만들 수 있는 도구다.
- 로그아웃 때 남으면 위험한 값은 사용자, Space, 기기 목록, 대시보드 상태다.
- MQTT와 Flutter BLE 연결 객체는 `ref.onDispose`에서 직접 닫는다.
- scope를 화면마다 만들면 캐시가 너무 자주 사라진다. 계정 전환 지점에만 크게 둔다.
