---
layout: post
title: "Flutter IoT Riverpod Provider 경계 - Flutter 클린 아키텍처에서 Repository를 숨기는 기준"
description: "Flutter IoT 앱에서 Riverpod Provider 경계를 잡아 Flutter 클린 아키텍처와 GetX Repository 패턴을 유지한 기준을 정리했다."
date: 2026-07-05
tags: [Flutter, Riverpod, CleanArchitecture, Repository패턴, IoT]
comments: true
share: true
---

![Flutter IoT Riverpod Provider boundary](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 앱 화면, Provider, Repository, 실제 IoT 장비 사이의 경계를 한 번 끊어서 봐야 한다.

Flutter IoT 앱에서 Riverpod을 넣는다고 Flutter 클린 아키텍처가 저절로 유지되지는 않았다. Flutter 스마트홈 대시보드에서 Flutter BLE 연결 상태와 MQTT 기기 상태를 Provider로 옮겼는데, 처음엔 `bleRepositoryProvider`, `mqttRepositoryProvider`, `deviceCardProvider`가 화면 파일 여기저기에 섞였다. GetX Repository 패턴으로 겨우 정리했던 경계가 Riverpod 전환 중 다시 흐려진 셈이다.

[클린 아키텍처 구조]({% post_url 2026-05-01-11-14-26-024690-clean-architecture-large-flutter-project-structure %})를 잡을 때는 Controller가 UseCase나 Repository를 직접 호출해도 위치가 비교적 눈에 보였다. Riverpod은 더 편한 대신 Provider가 많아지면 어디까지가 앱 상태이고 어디부터가 인프라 상태인지 금방 애매해진다. 내가 확인한 기준으론 화면이 Repository Provider를 직접 읽기 시작하면 위험 신호였다.

## Provider 경계를 세 단계로 나눴다

| 레이어 | Provider 예시 | 맡긴 일 |
|---|---|---|
| Infra | `mqttRepositoryProvider` | MQTT publish, subscribe, reconnect |
| App state | `deviceSessionProvider` | 연결 상태와 명령 대기 상태 관리 |
| View model | `deviceCardViewProvider` | 화면 문구, 버튼 활성화, 배지 색상 조합 |

핵심은 Widget이 Infra Provider를 모르게 하는 것이다. 화면은 MQTT인지 BLE인지 따지지 않고, 카드가 그릴 값만 읽는다. 반대로 Repository는 화면 문구를 모른다. "연결 끊김"이라는 텍스트를 Repository에서 만들기 시작하면 테스트도 번역도 같이 꼬였다.

화면이 읽는 Provider는 최대한 얇게 둔다.

```dart
final deviceCardViewProvider =
    Provider.family<DeviceCardView, String>((ref, deviceId) {
  final session = ref.watch(deviceSessionProvider(deviceId));

  return DeviceCardView(
    title: session.name,
    powerLabel: session.isOn ? '켜짐' : '꺼짐',
    canControl: session.isOnline && !session.commandPending,
    badge: session.isBleNear ? '근처' : '원격',
  );
});
```

처음엔 `deviceCardViewProvider` 안에서 `mqttRepositoryProvider`를 읽고 최신 상태를 가져오게 했다. 해보니 됐다. 문제는 카드가 rebuild될 때마다 상태 조회와 화면 조합이 같은 위치에서 일어났다는 점이다. MQTT 패킷 수신, BLE 근접 여부, 오프라인 캐시 복원이 한 Provider 안에 섞이니 로그를 봐도 어느 단계가 느린지 알기 어려웠다.

그래서 Repository 호출은 세션 Provider 아래로 내렸다.

```dart
final deviceSessionProvider =
    StreamProvider.family<DeviceSession, String>((ref, deviceId) {
  final repository = ref.watch(deviceRepositoryProvider);
  return repository.watchSession(deviceId);
});
```

이렇게 두면 `deviceCardViewProvider`는 순수 변환에 가깝다. 테스트도 쉬워진다. `DeviceSession`만 넣어주면 버튼이 비활성화되는지, BLE 근처 배지가 보이는지 확인할 수 있다. 실제 MQTT 브로커나 `flutter_blue_plus` 스캔 결과는 여기까지 올라오지 않는다.

주의할 점은 Provider 파일을 레이어별로만 쪼개면 오히려 찾기 어렵다는 것이다. `infra_providers.dart`, `state_providers.dart`처럼 나누면 처음엔 깔끔하지만 기기가 8개쯤 되면 관련 코드를 계속 이동해야 했다. 나는 기능 폴더 안에서 `data`, `application`, `presentation` 정도로만 나눴다. 너무 잘게 나누면 구조가 설계가 아니라 숨바꼭질이 된다.

짧게 남기면 이렇다. Flutter IoT Riverpod 구조에서 Widget은 View Provider만 읽고, View Provider는 Repository를 직접 호출하지 않는 편이 오래 버텼다. Repository는 통신을 맡고, 세션 Provider는 앱 상태를 만들고, 화면 Provider는 표시값만 조합한다. 이 선만 지켜도 GetX에서 Riverpod으로 넘어갈 때 Controller 비대화 문제가 Provider 난립으로 바뀌는 걸 꽤 막을 수 있다.
