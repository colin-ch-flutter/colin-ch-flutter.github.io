---
layout: post
title: "Flutter IoT Riverpod keepAlive - MQTT 세션을 화면 수명과 분리하기"
description: "Flutter IoT 앱에서 Riverpod keepAlive와 autoDispose 기준을 나눠 MQTT 세션이 화면 이동마다 끊기는 문제를 정리했다."
date: 2026-07-03
tags: [Flutter, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---
![Flutter IoT Riverpod keepAlive MQTT 세션 분리](https://images.unsplash.com/photo-1558346490-a72e53ae2d4f?w=800&q=80)

이 그림에서는 스마트홈 앱이 화면보다 긴 수명의 네트워크 세션을 가진다는 점을 봐야 한다.

Flutter IoT 앱에서 MQTT 세션은 화면 Provider처럼 쉽게 버리면 안 된다. Riverpod `autoDispose`를 습관처럼 붙이면 상세 화면을 나갔다 들어오는 것만으로 구독이 끊기고, 보일러 상태가 1~2초 늦게 복구되는 일이 생긴다. 화면 데이터는 짧게 살리고, MQTT 연결은 앱 세션 기준으로 오래 살리는 쪽이 더 안정적이다.

처음엔 모든 Provider에 `autoDispose`를 붙였다. 메모리 누수를 줄인다는 생각이었다. 실제로 등록 화면의 BLE 스캔 목록은 그렇게 하는 게 맞았다. 문제는 MQTT 연결 상태까지 같은 기준으로 묶은 뒤부터 나왔다. 탭을 바꿀 때마다 `disconnect -> connect -> subscribe`가 반복됐고, 사용자가 전원 버튼을 누른 직후 화면을 이동하면 publish 결과를 놓쳤다.

내가 잡은 기준은 단순하다.

| 상태 종류 | 권장 수명 | 이유 |
|---|---:|---|
| BLE 스캔 결과 | 화면 수명 | 등록 화면을 닫으면 스캔도 멈춰야 한다 |
| 대시보드 조회값 | Space/deviceId 수명 | 다른 집이나 기기로 바뀌면 다시 읽어야 한다 |
| MQTT 클라이언트 | 앱 로그인 세션 수명 | 화면 이동과 연결 수명은 다르다 |
| 토픽 구독 | Space 수명 | 집이 바뀌면 구독 토픽도 바뀐다 |

핵심은 `keepAlive`를 무조건 켜는 게 아니라, 세션 Provider에만 붙이는 것이다. 아래 코드는 MQTT 연결은 유지하고, 구독 Provider만 `autoDispose.family`로 분리한 형태다.

```dart
final mqttClientProvider = FutureProvider.autoDispose<MqttClient>((ref) async {
  final client = MqttClient('broker.example.com', 'mobile-app');

  ref.onDispose(() async {
    client.disconnect();
  });

  await client.connect();
  ref.keepAlive();

  return client;
});

final spaceSubscriptionProvider =
    Provider.autoDispose.family<void, String>((ref, spaceId) {
  final client = ref.watch(mqttClientProvider).requireValue;
  final topic = 'homes/$spaceId/devices/+/state';

  client.subscribe(topic);

  ref.onDispose(() {
    client.unsubscribe(topic);
  });
});
```

여기서 헷갈렸던 건 `ref.keepAlive()`를 호출했으니 절대 dispose되지 않는다고 착각한 부분이다. 실제로는 Provider가 살아 있는 동안 캐시를 유지하는 성격에 가깝고, 계정 로그아웃이나 ProviderScope 교체 같은 큰 단위에서는 정리될 수 있게 설계해야 한다. 그래서 로그아웃 시점에는 별도로 세션을 닫는 흐름을 둔다.

```dart
Future<void> logout(WidgetRef ref) async {
  final client = await ref.read(mqttClientProvider.future);
  client.disconnect();

  ref.invalidate(mqttClientProvider);
  ref.invalidate(selectedSpaceIdProvider);
}
```

또 하나의 실패는 구독까지 오래 살린 것이다. MQTT 클라이언트는 유지해도 `homes/a/devices/+/state` 구독은 Space A 전용이다. 사용자가 Space B로 바꿨는데 A 토픽이 남아 있으면, 서버에서 늦게 도착한 retained 메시지가 현재 화면에 섞일 수 있다. 이 버그는 네트워크가 느린 LTE 환경에서 더 잘 보였다.

운영하면서 본 증상은 대략 이랬다.

```text
화면 이동 전: MQTT connected, topic homes/a/devices/+/state
상세 화면 진입: provider disposed, mqtt disconnected
전원 명령 전송: publish failed, reconnecting
화면 복귀 후: 이전 retained message 표시
```

주의할 점은 `keepAlive`가 재연결 전략을 대신하지 않는다는 것이다. 브로커가 끊기거나 모바일 네트워크가 바뀌면 Repository 레벨에서 재시도와 에러 상태를 따로 관리해야 한다. Riverpod은 수명을 정리해줄 뿐, 통신 품질을 보장하지 않는다.

짧게 정리하면 이렇다.

- Flutter IoT 앱에서 MQTT 연결은 화면보다 길게 살아야 한다.
- `autoDispose`는 BLE 스캔, 화면 조회값, Space별 구독처럼 짧은 상태에 붙인다.
- `keepAlive`는 앱 로그인 세션과 함께 유지할 Provider에만 좁게 쓴다.
- 연결 수명과 토픽 구독 수명을 분리하면 화면 이동 중 publish 실패가 줄어든다.
