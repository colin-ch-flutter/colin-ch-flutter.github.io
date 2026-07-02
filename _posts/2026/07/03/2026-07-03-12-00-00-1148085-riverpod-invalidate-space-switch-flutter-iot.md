---
layout: post
title: "Flutter IoT Riverpod invalidate - Flutter 스마트홈 Space 전환 캐시 정리"
description: "Flutter IoT 앱에서 Riverpod ref.invalidate로 Flutter 스마트홈 Space 전환 시 BLE와 MQTT 기기 캐시가 섞이는 문제를 정리했다."
date: 2026-07-03
tags: [Flutter, IoT, 스마트홈, BLE, MQTT]
comments: true
share: true
---
![Flutter IoT Riverpod invalidate 스마트홈 Space 전환 캐시 정리](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

Flutter IoT 앱에서 Flutter 스마트홈 Space를 바꿀 때는 Riverpod Provider 캐시를 의도적으로 버려야 한다. BLE 스캔 목록, MQTT 구독 상태, 대시보드 데이터가 이전 Space 기준으로 남아 있으면 사용자는 다른 집을 선택했는데 앱은 예전 보일러 상태를 잠깐 보여준다. `ref.invalidate`를 화면 새로고침 버튼에만 쓰면 이 문제를 놓치기 쉽다.

처음엔 `spaceId`만 바꾸면 Provider들이 알아서 다시 빌드될 줄 알았다. 반은 맞고 반은 틀렸다. `family(spaceId)`로 만든 Provider는 새 키로 다시 읽히지만, 앱 전체에서 공유하던 BLE 연결 상태나 MQTT 세션 Provider는 그대로 살아 있었다. 해보니 됐다가 아니라, 운 좋게 화면이 다시 그려졌을 뿐이었다.

GetX 시절에는 Space 변경 로직이 Controller 여러 개에 흩어졌다. 그래서 [`GetX 상태관리`]({% post_url 2026-05-02-09-21-39-037035-getx-state-management-routing-flutter-iot %})를 쓸 때 편하긴 했지만, 프로젝트가 커질수록 어떤 Controller를 비워야 하는지 헷갈렸다. Riverpod에서는 전환 이벤트 한 곳에서 버릴 Provider를 명시하는 쪽이 더 낫다.

Space 변경 직후에는 화면 데이터 Provider를 무효화하고, 통신 세션은 끊을지 유지할지 따로 판단한다. 같은 계정의 집만 바꾸는 상황이라면 MQTT 클라이언트 자체를 매번 재생성하지 않아도 된다. 대신 토픽 구독은 반드시 새 Space 기준으로 다시 잡아야 한다.

```dart
final selectedSpaceIdProvider = StateProvider<String?>((ref) => null);

final dashboardProvider =
    FutureProvider.family<DashboardState, String>((ref, spaceId) async {
  final repository = ref.watch(homeRepositoryProvider);
  return repository.fetchDashboard(spaceId);
});

Future<void> switchSpace(WidgetRef ref, String nextSpaceId) async {
  final previousSpaceId = ref.read(selectedSpaceIdProvider);
  if (previousSpaceId == nextSpaceId) return;

  ref.read(selectedSpaceIdProvider.notifier).state = nextSpaceId;

  if (previousSpaceId != null) {
    ref.invalidate(dashboardProvider(previousSpaceId));
    ref.invalidate(deviceListProvider(previousSpaceId));
  }

  ref.invalidate(dashboardProvider(nextSpaceId));
  ref.invalidate(deviceListProvider(nextSpaceId));
}
```

여기서 중요한 건 `mqttConnectionProvider`까지 습관적으로 invalidate하지 않는 것이다. 연결 Provider를 버리면 앱은 Space 전환마다 소켓을 끊고 다시 붙는다. 실제 기기에서는 이 짧은 순간에 명령 publish가 실패하거나 retained 메시지를 늦게 받아서 상태가 튀었다. 나는 이걸 BLE 문제로 착각하고 한참 로그를 봤는데, 원인은 Provider 수명 설계였다.

MQTT 구독은 별도 Provider로 빼는 편이 깔끔했다. 연결은 앱 생명주기 기준으로 유지하고, Space별 토픽 구독만 `spaceId`에 묶는다.

```dart
final spaceMqttSubscriptionProvider =
    Provider.autoDispose.family<void, String>((ref, spaceId) {
  final mqtt = ref.watch(mqttClientProvider);
  mqtt.subscribe('homes/$spaceId/devices/+/state');

  ref.onDispose(() {
    mqtt.unsubscribe('homes/$spaceId/devices/+/state');
  });
});
```

BLE도 비슷하다. 등록 화면에서 본 주변 기기 목록은 Space 전환과 직접 관계가 없을 수 있다. 반대로 이미 등록된 기기와 BLE MAC을 매핑한 캐시는 Space가 바뀌면 버려야 한다. 서버 deviceId와 BLE MAC을 섞어 쓰는 앱이라면 이 기준을 더 엄격하게 잡아야 한다.

주의할 점은 `invalidate`를 많이 호출한다고 상태 관리가 좋아지는 건 아니라는 것이다. Provider를 버리는 기준은 사용자, Space, deviceId처럼 데이터 소유권이 바뀌는 순간이어야 한다. 단순 로딩 실패 재시도와 계정/공간 전환은 다른 문제다.

짧게 정리하면 이렇다.

- Flutter IoT 앱의 Space 전환은 화면 이동이 아니라 데이터 소유권 변경이다.
- Riverpod `family` Provider는 이전 키와 다음 키를 구분해서 invalidate한다.
- MQTT 연결 자체와 Space별 토픽 구독은 수명을 분리한다.
- BLE 등록 캐시와 서버 기기 캐시를 같은 기준으로 지우면 버그가 숨어든다.
