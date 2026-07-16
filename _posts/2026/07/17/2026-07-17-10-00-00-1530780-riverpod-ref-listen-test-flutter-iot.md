---
layout: post
title: "Flutter IoT Riverpod ref.listen 테스트 - BLE·MQTT 알림 중복 실행 막기"
description: "Flutter IoT 앱에서 Riverpod ref.listen과 ProviderContainer.listen을 테스트해 BLE 연결 알림과 MQTT 상태 메시지의 중복 실행을 막는 기준을 정리했다."
date: 2026-07-17
tags: [Flutter, Dart, Riverpod, BLE, MQTT, IoT, 테스트]
comments: true
share: true
---
![Flutter IoT Riverpod ref.listen 테스트와 BLE MQTT 상태 알림](https://images.unsplash.com/photo-1558002038-1055907df827?w=1200&q=80)

이 그림에서 볼 부분은 기기 상태가 바뀌는 통신 흐름과 화면 알림을 분리하는 지점이다.

Flutter IoT 앱에서 Riverpod `ref.listen`은 상태가 바뀌었을 때 한 번 실행할 일에 맞다. BLE 연결 끊김 SnackBar나 MQTT ACK 실패 진동이 대표적이다. 처음엔 화면이 보이는지만 테스트하면 된다고 생각했는데, 실제 버그는 알림 중복과 이전 화면 listener 잔류에서 생겼다.

## 화면 rebuild와 부수 효과는 분리해야 한다

`ref.watch`는 위젯을 다시 그리고, `ref.listen`은 상태 변화 콜백을 실행한다. 두 역할을 섞으면 MQTT 메시지 한 번에 알림이 중복될 수 있다.

| 목적 | 적합한 방식 | 테스트에서 볼 것 |
|---|---|---|
| 온도·연결 상태 표시 | `ref.watch` | 위젯이 올바른 값을 표시하는가 |
| 연결 끊김 SnackBar | `ref.listen` | 변화 1회에 알림 1회인가 |
| Repository 호출 | Notifier 내부 | 호출 횟수와 인자가 맞는가 |

실제 BLE와 MQTT 구현은 Fake로 교체하고, 상태를 바꾼 뒤 `container.listen`으로 변화를 직접 관찰한다. [ProviderContainer override 패턴]({% post_url 2026-07-16-16-00-00-1518435-riverpod-providercontainer-overrides-test-flutter-iot %})과 함께 쓰기 좋다.

## ProviderContainer.listen으로 중복 여부를 검증한다

테스트에서는 UI 없이 Provider의 이전 값과 새 값을 비교할 수 있다. 연결 상태를 두 번 바꿔도 `disconnected` 진입 시 한 번만 기록하는 예제다.

```dart
enum ConnectionState { connected, disconnected }

final connectionProvider = StateProvider<ConnectionState>(
  (ref) => ConnectionState.connected,
);

test('BLE 연결 끊김 알림은 상태 변화마다 한 번만 기록한다', () {
  final container = ProviderContainer();
  addTearDown(container.dispose);
  final events = <ConnectionState>[];

  final subscription = container.listen<ConnectionState>(
    connectionProvider,
    (previous, next) {
      if (next == ConnectionState.disconnected &&
          previous != ConnectionState.disconnected) {
        events.add(next);
      }
    },
    fireImmediately: false,
  );
  addTearDown(subscription.close);

  final state = container.read(connectionProvider.notifier);
  state.state = ConnectionState.disconnected;
  state.state = ConnectionState.disconnected;
  state.state = ConnectionState.connected;

  expect(events, [ConnectionState.disconnected]);
});
```

여기서 `previous`를 확인하지 않으면 같은 끊김 상태를 다시 대입하는 코드에도 알림이 실행된다. MQTT 재연결 로직이 여러 경로에서 `disconnected`를 발행하는 구조라면 이 조건이 필요하다.

## 화면 생명주기에서 생긴 실수

내 경우 대시보드와 기기 상세 화면이 같은 `connectionProvider`를 각각 듣고 있었다. 화면을 push했다가 pop하는 동안 이전 listener가 남아 연결 복구 메시지가 두 번 표시됐다. 상태 변화와 화면 테스트를 나누고, 화면 제거 뒤 Stream 구독도 닫히는지 검증했다.

`fireImmediately: true`를 쓰면 초기 상태도 콜백에 전달된다. 초기 연결 알림이 목적이 아니라면 `false`가 안전하다. 테스트마다 `ProviderContainer`를 새로 만들고 `dispose`해야 이전 MQTT Stream이 다음 테스트에 남지 않는다.

`ref.watch`는 표시, `ref.listen`은 부수 효과다. `ProviderContainer.listen`에서 이전 값과 새 값을 함께 검사하면 BLE 끊김과 MQTT 재연결 알림 중복을 빠르게 잡을 수 있다.
