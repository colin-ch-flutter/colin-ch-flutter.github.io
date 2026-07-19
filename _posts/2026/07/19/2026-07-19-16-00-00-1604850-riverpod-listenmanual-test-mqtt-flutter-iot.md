---
layout: post
title: "Flutter IoT Riverpod listenManual 테스트 - MQTT 알림 중복 구독 막기"
description: "Flutter IoT 앱에서 Riverpod listenManual로 MQTT 알림 부작용을 화면 생명주기와 분리하고, 테스트에서 중복 구독과 해제를 검증하는 방법을 정리했다."
date: 2026-07-19
tags: [Flutter, Riverpod, MQTT, mqtt5_client, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod listenManual 테스트와 MQTT 알림](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Flutter IoT 앱에서 Riverpod `listenManual`은 MQTT 알림처럼 화면이 보일 때만 연결하고 사라질 때 해제해야 하는 부작용을 테스트하기 좋다. `ref.listen`은 위젯 생명주기에 자연스럽게 붙지만, 앱이 백그라운드에서 돌아오거나 테스트에서 ProviderContainer를 직접 만들 때 등록 시점을 통제하기 어렵다. 처음엔 `ref.listen`이면 충분하다고 생각했는데, 알림 콜백이 두 번 실행되는 로그를 보고 구독 핸들을 직접 관리하게 됐다.

## 문제가 생긴 상황

보일러 목표 온도가 바뀌면 MQTT 토픽을 구독하고 SnackBar를 띄우는 화면이 있었다. 화면을 열 때마다 아래 흐름이 반복됐다.

| 상황 | 실제 증상 | 원인 |
|---|---|---|
| 대시보드 재진입 | 같은 알림이 2번 표시됨 | 이전 listener가 남음 |
| 테스트를 여러 번 실행 | 호출 횟수가 실행 순서에 따라 달라짐 | Container와 구독 수명 혼합 |
| 로그아웃 후 재로그인 | 이전 집 알림이 새 화면에 도착함 | MQTT listener 미해제 |

화면에서 [`ref.listen`]({% post_url 2026-07-17-10-00-00-1530780-riverpod-ref-listen-test-flutter-iot %})을 여러 곳에 두고 `fireImmediately` 옵션으로 초기 상태까지 받게 한 것이 발단이었다. MQTT 연결 자체는 한 번인데, 상태 변화를 받는 Dart listener가 화면마다 추가되고 있었다.

## listenManual로 구독 수명을 명시한다

`listenManual`은 `ProviderSubscription`을 반환하므로 어느 시점에 열고 닫는지 코드로 드러난다. 알림 전용 Notifier를 두고, Widget은 구독과 해제만 담당하게 했다.

```dart
final mqttAlertProvider = NotifierProvider<MqttAlertNotifier, String?>
    (MqttAlertNotifier.new);

class MqttAlertNotifier extends Notifier<String?> {
  @override
  String? build() => null;

  void received(String message) => state = message;
}

class AlertPage extends ConsumerStatefulWidget {
  const AlertPage({super.key});

  @override
  ConsumerState<AlertPage> createState() => _AlertPageState();
}

class _AlertPageState extends ConsumerState<AlertPage> {
  ProviderSubscription<String?>? _subscription;

  @override
  void initState() {
    super.initState();
    _subscription = ref.listenManual<String?>(
      mqttAlertProvider,
      (previous, next) {
        if (!mounted || next == null || next == previous) return;
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next)),
        );
      },
    );
  }

  @override
  void dispose() {
    _subscription?.close();
    super.dispose();
  }
}
```

핵심은 `listenManual`을 썼다는 사실보다 `ProviderSubscription`을 필드로 보관하고 `dispose`에서 닫는 데 있다. `ref`가 자동으로 정리해 주는 상황에 기대면 Widget 테스트에서 직접 만든 수명과 실제 앱 수명이 다르게 움직일 수 있다. MQTT 수신 서비스는 `mqttAlertProvider.notifier.received(message)`만 호출하고, SnackBar 같은 UI 부작용은 화면에 남겨 책임도 분리했다.

## 테스트에서는 호출 횟수와 해제를 같이 본다

상태가 바뀌었는지만 확인하면 중복 listener를 잡지 못한다. 테스트용 `ProviderContainer`에서 구독을 두 번 열었을 때 콜백이 두 번 호출되는지, 하나를 닫으면 한 번만 호출되는지를 검증했다.

```dart
test('listenManual 구독을 닫으면 MQTT 알림 콜백이 중복 실행되지 않는다', () {
  final container = ProviderContainer();
  addTearDown(container.dispose);

  var callbackCount = 0;
  final first = container.listen<String?>(
    mqttAlertProvider,
    (_, next) {
      if (next != null) callbackCount++;
    },
    fireImmediately: false,
  );
  final second = container.listen<String?>(
    mqttAlertProvider,
    (_, next) {
      if (next != null) callbackCount++;
    },
    fireImmediately: false,
  );

  container.read(mqttAlertProvider.notifier).received('보일러 켜짐');
  expect(callbackCount, 2);

  second.close();
  container.read(mqttAlertProvider.notifier).received('보일러 꺼짐');
  expect(callbackCount, 3);

  first.close();
});
```

이 테스트는 `listenManual` 자체의 동작보다 구독이 여러 개일 때 부작용이 배수로 실행된다는 점을 보여준다. 실제 Widget 테스트에서는 `pumpWidget`으로 화면을 두 번 교체한 뒤 Fake MQTT 서비스의 이벤트를 발행하고 SnackBar 개수를 확인하면 된다. `pumpAndSettle`만 믿으면 SnackBar 애니메이션 타이밍 때문에 간헐적으로 실패했으므로, 이벤트 발행과 `pump` 사이를 분리했다.

## 정리

- `ref.listen`은 간단한 상태 반응에, `listenManual`은 수명을 직접 관리해야 하는 UI 부작용에 맞는다.
- `ProviderSubscription`을 저장하고 `dispose`에서 반드시 `close`한다.
- MQTT 테스트는 상태값보다 알림 콜백 호출 횟수와 해제 뒤 호출 여부를 검증한다.
- `ProviderContainer` 테스트에는 `addTearDown(container.dispose)`를 넣어 테스트 간 listener가 남지 않게 한다.

솔직하게 정리하면, `listenManual`은 코드를 줄여주는 API가 아니다. 대신 Flutter 스마트홈 화면이 사라졌을 때 MQTT 알림도 같이 사라져야 한다는 수명 규칙을 코드와 테스트에 남겨준다. 이 규칙이 있어야 재진입과 로그아웃이 섞여도 중복 알림을 추적하기 쉬웠다.
