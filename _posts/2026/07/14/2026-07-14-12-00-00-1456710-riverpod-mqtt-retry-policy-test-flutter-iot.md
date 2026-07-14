---
layout: post
title: "Flutter IoT Riverpod MQTT 재시도 - 연결 끊김을 무한 반복하지 않는 테스트 기준"
description: "Flutter IoT 앱에서 Riverpod으로 MQTT 연결 끊김 재시도 횟수와 지수 백오프를 제어하고, fakeAsync로 Flutter MQTT 재연결 정책을 검증하는 방법을 정리했다."
date: 2026-07-14
tags: [Flutter, Riverpod, MQTT, IoT, 스마트홈, Dart]
comments: true
share: true
---
![Flutter IoT Riverpod MQTT 재시도 정책](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

이 그림에서 볼 부분은 연결이 끊겼을 때 즉시 무한 반복하는 대신, 정해진 횟수와 간격으로 복구를 시도하는 흐름이다.

Flutter IoT 앱의 MQTT 연결 끊김은 `retry()` 한 줄로 끝내면 안 된다. Riverpod Provider가 다시 만들어질 때마다 재시도 카운터가 0으로 돌아가면 네트워크가 없는 동안 연결 요청이 계속 쌓인다. 반대로 너무 보수적으로 막으면 사용자가 앱을 다시 열 때까지 보일러 상태가 갱신되지 않는다. 내가 사용한 기준은 최대 4회, 1초·2초·4초·8초의 지수 백오프다. 4회 실패하면 `needsReconnect` 상태를 화면에 보여주고 자동 시도는 멈춘다.

## 처음 실패한 가정

처음엔 MQTT Repository의 `connect()` 안에서 `while` 문으로 재시도했다. 실제 기기에서는 동작하는 것처럼 보였지만 문제가 두 가지 있었다.

| 상황 | while 내부 재시도의 문제 | Provider 상태로 분리한 결과 |
|---|---|---|
| 앱이 백그라운드로 이동 | Future가 계속 살아 있음 | 생명주기 이벤트에서 중단 가능 |
| 네트워크 없음 | 타이머와 연결 요청이 누적됨 | 최대 횟수 초과 시 종료 |
| 화면 표시 | 연결 중인지 알기 어려움 | `AsyncValue`로 즉시 표현 |

그래서 Repository는 한 번의 연결만 담당하고, 재시도 횟수와 대기 시간은 Riverpod `AsyncNotifier`가 관리하도록 나눴다. 연결 실패를 곧바로 throw하는 것도 중요하다. 예외를 삼키면 Provider는 성공한 것으로 남고, UI는 오래된 보일러 상태를 정상값처럼 보여준다.

## 재시도 정책을 상태로 둔다

아래 코드는 실제 MQTT 클라이언트 대신 Repository 인터페이스를 사용한다. 테스트에서는 이 자리에 `FakeMqttRepository`를 넣어 연결 성공과 실패를 원하는 순서로 만들 수 있다.

```dart
enum MqttConnectionState { idle, connecting, connected, needsReconnect }

class MqttStatus {
  const MqttStatus(this.state, {this.attempt = 0});

  final MqttConnectionState state;
  final int attempt;
}

final mqttRepositoryProvider = Provider<MqttRepository>((ref) {
  return AwsMqttRepository();
});

final mqttStatusProvider = AsyncNotifierProvider<MqttStatusNotifier, MqttStatus>(
  MqttStatusNotifier.new,
);

class MqttStatusNotifier extends AsyncNotifier<MqttStatus> {
  @override
  Future<MqttStatus> build() async =>
      const MqttStatus(MqttConnectionState.idle);

  Future<void> connectWithRetry() async {
    final repository = ref.read(mqttRepositoryProvider);
    state = const AsyncData(MqttStatus(MqttConnectionState.connecting));

    for (var attempt = 1; attempt <= 4; attempt++) {
      try {
        await repository.connect();
        state = const AsyncData(MqttStatus(MqttConnectionState.connected));
        return;
      } catch (_) {
        if (attempt == 4) {
          state = AsyncData(const MqttStatus(
            MqttConnectionState.needsReconnect,
            attempt: 4,
          ));
          return;
        }

        state = AsyncData(MqttStatus(
          MqttConnectionState.connecting,
          attempt: attempt,
        ));
        await Future<void>.delayed(Duration(seconds: 1 << (attempt - 1)));
      }
    }
  }
}
```

여기서 `needsReconnect`를 별도 상태로 둔 이유는 실패와 재시도 중을 같은 에러로 표시하면 안 되기 때문이다. 2번째 시도 직전에는 로딩 인디케이터와 “재연결 중 2/4”를 보여주고, 4회가 끝난 뒤에는 수동 재연결 버튼을 보여주는 식이다.

## `fakeAsync`로 시간과 호출 횟수를 고정한다

실제 1초, 2초를 기다리는 테스트는 CI에서 느리고 흔들린다. `fakeAsync`를 쓰면 타이머를 직접 진행시키면서 MQTT 연결 호출 횟수와 최종 상태를 확인할 수 있다.

```dart
test('MQTT 연결이 4번 실패하면 자동 재시도를 멈춘다', () {
  fakeAsync((async) {
    final fake = FakeMqttRepository(alwaysFail: true);
    final container = ProviderContainer(
      overrides: [mqttRepositoryProvider.overrideWithValue(fake)],
    );
    addTearDown(container.dispose);

    container.read(mqttStatusProvider.notifier).connectWithRetry();
    async.flushMicrotasks();
    expect(fake.connectCount, 1);

    async.elapse(const Duration(seconds: 1));
    async.flushMicrotasks();
    async.elapse(const Duration(seconds: 2));
    async.flushMicrotasks();
    async.elapse(const Duration(seconds: 4));
    async.flushMicrotasks();
    async.elapse(const Duration(seconds: 8));
    async.flushMicrotasks();

    expect(fake.connectCount, 4);
    expect(
      container.read(mqttStatusProvider).value?.state,
      MqttConnectionState.needsReconnect,
    );
  });
});
```

처음 이 테스트를 만들 때 `elapse(Duration(seconds: 15))` 한 번으로 끝내려 했다. 그런데 테스트가 타이머를 한꺼번에 진행하면서 중간 상태를 검증하지 못했다. 연결 호출이 실제로 1초, 2초, 4초 뒤에 발생하는지 구간별로 나눠 확인하는 편이 실패 원인을 찾기 좋다. 성공하는 Fake를 넣을 때는 2번째 호출에서 성공하도록 만들어 `connected` 전환과 이후 추가 호출이 없는지도 확인해야 한다.

## 운영에서 조심할 부분

`connectWithRetry()`가 여러 번 동시에 호출되지 않게 해야 한다. 앱 포그라운드 이벤트와 사용자의 재연결 버튼이 같은 순간 들어오면 연결 시도가 두 묶음으로 실행될 수 있다. `bool _connecting` 가드나 `CancelableOperation`으로 이전 작업을 취소하는 경계를 두는 게 안전하다.

또한 백오프는 네트워크 장애를 해결하는 기능이 아니다. MQTT 브로커가 인증서 만료나 AWS IoT SigV4 오류로 거절한 경우에는 4번 재시도해도 성공하지 않는다. 인증 실패와 일시적인 소켓 단절을 분류해서, 전자는 즉시 로그아웃·재인증 흐름으로 보내야 한다.

짧게 정리하면 다음 기준이다.

- Repository는 한 번의 MQTT 연결만 담당한다.
- Riverpod 상태에 재시도 중과 최종 실패를 분리한다.
- 최대 횟수와 백오프 간격을 숫자로 고정하고 `fakeAsync`로 검증한다.
- 인증 오류는 재시도하지 않고, 동시 연결과 Provider dispose를 별도로 막는다.
