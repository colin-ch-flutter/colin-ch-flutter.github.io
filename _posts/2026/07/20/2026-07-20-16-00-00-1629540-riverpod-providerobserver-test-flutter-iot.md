---
layout: post
title: "Flutter IoT Riverpod ProviderObserver 테스트 - MQTT 상태 변화와 에러 추적"
description: "Flutter IoT 앱에서 Riverpod ProviderObserver로 BLE·MQTT 상태 변화와 provider 에러를 기록하고, 테스트에서 연결 끊김 이벤트와 중복 로그를 검증하는 방법을 정리했다."
date: 2026-07-20
tags: [Flutter, Dart, Riverpod, MQTT, BLE, IoT, 테스트]
comments: true
share: true
---

![Flutter IoT Riverpod ProviderObserver 테스트](/assets/images/flutter-iot-riverpod-providerobserver-test.png)

이 그림에서 볼 부분은 스마트홈 화면의 상태 변화가 기기와 클라우드 이벤트 로그로 이어진다는 점이다.

Flutter IoT 앱에서 MQTT가 끊겼는데 화면에는 계속 `AsyncData`가 남아 있거나, BLE 재연결 실패가 조용히 사라지는 경우가 있다. Provider 내부에 로그를 잔뜩 넣으면 원인은 찾을 수 있지만 테스트가 로그 구현에 묶인다. `ProviderObserver`를 앱 루트에 붙이고, 테스트에서는 이벤트를 수집하는 Fake Observer를 주입하면 상태 변화와 에러를 별도로 검증할 수 있다.

## 로그를 Notifier 밖으로 뺀 이유

처음엔 `AsyncNotifier`의 `try/catch` 안에서 `debugPrint()`를 호출했다. 개발 중에는 편했지만 문제가 두 가지 생겼다. 같은 MQTT 재연결이 두 번 실행됐는지 테스트할 방법이 없었고, 운영 로그를 Crashlytics로 보낼 때 모든 Notifier를 다시 수정해야 했다.

Observer가 있으면 상태 관리 코드는 상태만 바꾸고, 관찰 코드는 한 곳에 모인다.

| 확인 대상 | Notifier 내부 로그 | ProviderObserver |
| --- | --- | --- |
| 상태 전이 | 직접 문자열을 만들어야 함 | provider와 이전·이후 값 수집 |
| 예외 | catch 블록마다 처리 | `providerDidFail` 한 곳에서 수집 |
| 중복 이벤트 | 호출 횟수 추적이 번거로움 | 테스트 이벤트 목록으로 검증 |

## 테스트용 Observer를 만든다

`didUpdateProvider`는 provider의 값이 바뀔 때 호출된다. 테스트에서는 문자열 로그보다 provider 이름과 값의 타입을 저장하는 편이 비교하기 쉽다.

```dart
class ProviderEvent {
  const ProviderEvent({
    required this.provider,
    required this.previous,
    required this.current,
  });

  final Object provider;
  final Object? previous;
  final Object? current;
}

class RecordingObserver extends ProviderObserver {
  final updates = <ProviderEvent>[];
  final failures = <Object>[];

  @override
  void didUpdateProvider(
    ProviderBase<Object?> provider,
    Object? previousValue,
    Object? newValue,
    ProviderContainer container,
  ) {
    updates.add(ProviderEvent(
      provider: provider,
      previous: previousValue,
      current: newValue,
    ));
  }

  @override
  void providerDidFail(
    ProviderBase<Object?> provider,
    Object error,
    StackTrace stackTrace,
    ProviderContainer container,
  ) {
    failures.add(error);
  }
}
```

`ProviderEvent`에 실제 값 전체를 저장하면 MQTT payload나 토큰이 테스트 출력에 노출될 수 있다. 운영 코드에서 재사용할 Observer라면 기기 ID, provider runtimeType, 상태 종류 정도만 남기는 게 안전하다.

## MQTT 연결 상태 변화를 검증한다

관찰할 상태는 복잡한 MQTT 클라이언트가 아니라 앱이 화면에 보여주는 도메인 상태로 제한한다. 그래야 `mqtt5_client` API가 바뀌어도 이 테스트는 흔들리지 않는다.

```dart
enum MqttStatus { disconnected, connecting, connected }

final mqttStatusProvider =
    StateProvider<MqttStatus>((ref) => MqttStatus.disconnected);

void applyMqttStatus(WidgetRef ref, MqttStatus status) {
  ref.read(mqttStatusProvider.notifier).state = status;
}
```

이제 실제 MQTT 서비스 대신 `ProviderContainer`에서 상태를 바꾸고 Observer가 전이를 기록하는지만 확인한다.

```dart
void main() {
  test('MQTT 연결 전이를 Observer가 순서대로 기록한다', () {
    final observer = RecordingObserver();
    final container = ProviderContainer(observers: [observer]);
    addTearDown(container.dispose);

    final notifier = container.read(mqttStatusProvider.notifier);
    notifier.state = MqttStatus.connecting;
    notifier.state = MqttStatus.connected;
    notifier.state = MqttStatus.disconnected;

    final events = observer.updates
        .where((event) => event.provider == mqttStatusProvider)
        .toList();

    expect(events.map((event) => event.current), [
      MqttStatus.connecting,
      MqttStatus.connected,
      MqttStatus.disconnected,
    ]);
  });
}
```

여기서 `provider == mqttStatusProvider` 비교가 중요하다. 하나의 Container에서 BLE provider와 MQTT provider를 같이 읽으면 Observer 이벤트가 섞이기 때문이다. 처음에는 `updates.length`만 검사했다가 BLE 초기화 이벤트까지 포함되어 테스트가 간헐적으로 실패했다.

## provider 에러와 중복 재시도를 잡는다

`providerDidFail`는 provider가 에러 상태가 되었을 때 호출된다. MQTT 재연결 로직에서 같은 예외를 두 번 올리는 실수를 방지하려면 실패 횟수와 에러 타입을 함께 확인한다.

```dart
final mqttHealthProvider = FutureProvider<String>((ref) async {
  throw StateError('broker unavailable');
});

test('MQTT broker 오류가 한 번만 기록된다', () async {
  final observer = RecordingObserver();
  final container = ProviderContainer(observers: [observer]);
  addTearDown(container.dispose);

  final sub = container.listen(mqttHealthProvider, (_, __) {});
  addTearDown(sub.close);

  await expectLater(
    container.read(mqttHealthProvider.future),
    throwsA(isA<StateError>()),
  );

  expect(observer.failures, hasLength(1));
  expect(observer.failures.single, isA<StateError>());
});
```

다만 provider를 읽는 것만으로는 `autoDispose` provider가 바로 정리될 수 있다. `listen`으로 관찰을 유지하고, 테스트가 끝날 때 구독과 Container를 모두 닫아야 한다. `addTearDown` 순서를 의식하지 않으면 실제 앱에서는 없던 리소스 누수가 테스트 로그에 남는다.

## 적용할 때 헷갈린 지점

Observer는 디버깅과 테스트에 유용하지만 모든 상태를 무조건 기록하는 전역 로거는 아니다. 화면이 새로 그려질 때마다 값이 바뀌는 임시 UI provider까지 수집하면 로그가 폭발한다. 내 기준으로는 BLE 연결, MQTT 세션, 인증처럼 장애 원인을 설명하는 provider만 등록 대상에 넣는다.

```text
도메인 상태 변경 → ProviderObserver
                 ├─ 개발 환경: 콘솔 로그
                 ├─ 테스트: RecordingObserver
                 └─ 운영 환경: 개인정보를 뺀 구조화 이벤트
```

요점은 세 가지다.

- `ProviderObserver`로 상태 관찰과 Notifier 구현을 분리한다.
- 테스트에서는 provider를 필터링해 BLE·MQTT 이벤트가 섞이지 않게 한다.
- `providerDidFail`는 실패 횟수와 에러 타입을 함께 검증하고, 구독 수명은 직접 정리한다.
