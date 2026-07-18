---
layout: post
title: "Flutter IoT Riverpod overrideWithBuild 테스트 - MQTT 초기 상태 주입과 명령 검증"
description: "Flutter IoT 앱에서 Riverpod 3.0 overrideWithBuild와 ProviderContainer.test로 MQTT 연결 상태만 바꿔 주입하고, 원래 Notifier 명령 로직을 Fake 없이 검증하는 방법을 정리했다."
date: 2026-07-18
tags: [Flutter, Dart, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod overrideWithBuild 테스트와 MQTT 상태 전이](/images/2026-07-18-riverpod-overridewithbuild-test-mqtt.png)

그림에서 볼 부분은 테스트가 MQTT 연결 자체를 흉내 내는 게 아니라, 제어 화면이 시작할 상태만 주입하고 실제 명령 메서드의 상태 전이는 그대로 실행한다는 점이다.

Flutter IoT 앱에서 Riverpod `overrideWithBuild`를 쓰면 MQTT Repository를 통째로 Mock으로 바꾸지 않고도 Notifier 테스트를 짧게 만들 수 있다. 처음엔 연결 Fake, 응답 Fake, Notifier Mock을 모두 만들었다. 해보니 테스트가 실제 앱 코드보다 Mock 코드에 더 끌려갔다. Riverpod 3.0의 `NotifierProvider.overrideWithBuild`는 `build()` 결과만 바꾸고, Notifier에 정의한 `turnOn()` 같은 메서드는 원래 구현을 유지한다.

## MQTT 상태를 반환하는 Notifier

테스트 대상은 보일러 연결 상태와 마지막 명령을 갖는 간단한 Notifier다. 실제 앱에서는 Repository가 MQTT 메시지를 발행하지만, 여기서는 명령 호출 횟수나 UI 상태 전이를 검증하는 데 집중한다.

```dart
enum MqttStatus { disconnected, connected, publishing, error }

class BoilerState {
  const BoilerState({
    required this.status,
    required this.isOn,
    this.lastCommand,
  });

  final MqttStatus status;
  final bool isOn;
  final String? lastCommand;

  BoilerState copyWith({
    MqttStatus? status,
    bool? isOn,
    String? lastCommand,
  }) {
    return BoilerState(
      status: status ?? this.status,
      isOn: isOn ?? this.isOn,
      lastCommand: lastCommand ?? this.lastCommand,
    );
  }
}

class BoilerNotifier extends Notifier<BoilerState> {
  @override
  BoilerState build() => const BoilerState(
        status: MqttStatus.disconnected,
        isOn: false,
      );

  Future<void> turnOn() async {
    state = state.copyWith(
      status: MqttStatus.publishing,
      lastCommand: 'boiler/on',
    );

    // 실제 구현에서는 Repository.publish()를 호출한다.
    await Future<void>.delayed(Duration.zero);
    state = state.copyWith(
      status: MqttStatus.connected,
      isOn: true,
    );
  }
}

final boilerProvider =
    NotifierProvider<BoilerNotifier, BoilerState>(BoilerNotifier.new);
```

`build()`는 기본값을 반환하지만, 테스트에서는 연결된 보일러 상태에서 시작하고 싶을 수 있다. 이때 Notifier 전체를 교체하면 `turnOn()`까지 테스트용 구현이 되어 버린다. `overrideWithBuild`는 이 경계를 정확히 나눈다.

## build만 바꿔서 원래 명령을 실행한다

아래 테스트는 초기 상태를 `connected`로 주입한 뒤, 실제 `turnOn()`을 호출한다. `ProviderContainer.test()`는 테스트 종료 시 컨테이너를 정리해 주므로 별도 `tearDown` 누락도 줄어든다.

```dart
test('연결된 MQTT 상태에서 turnOn 명령이 실제 구현대로 전이된다', () async {
  final container = ProviderContainer.test(
    overrides: [
      boilerProvider.overrideWithBuild((ref, notifier) {
        return const BoilerState(
          status: MqttStatus.connected,
          isOn: false,
        );
      }),
    ],
  );

  expect(
    container.read(boilerProvider).status,
    MqttStatus.connected,
  );

  await container.read(boilerProvider.notifier).turnOn();

  final result = container.read(boilerProvider);
  expect(result.status, MqttStatus.connected);
  expect(result.isOn, isTrue);
  expect(result.lastCommand, 'boiler/on');
});
```

여기서 헷갈렸던 건 두 번째 인자인 `notifier`를 꼭 써야 한다고 생각한 부분이다. 초기 상태 계산에 외부 설정이 필요하지 않으면 `ref`와 `notifier`를 사용하지 않아도 된다. 중요한 건 override가 `turnOn()`을 가리지 않는다는 점이다. 그래서 이 테스트는 “초기 연결 상태에서 실제 명령 메서드가 정상적으로 동작하는가”를 확인한다.

## AsyncNotifier 테스트와 구분할 점

`overrideWithBuild`는 상태 초기화 테스트에 특히 잘 맞는다. MQTT 연결·재시도 자체의 성공과 실패를 검증할 때는 기존처럼 Fake Repository를 주입하는 편이 낫다. 네트워크 결과를 `build()`에 숨기면 실패 원인과 재시도 횟수를 검증할 수 없기 때문이다.

| 검증하려는 것 | 적합한 방법 | 이유 |
| --- | --- | --- |
| 연결된 상태에서 버튼 명령 | `overrideWithBuild` | 원래 Notifier 메서드 유지 |
| MQTT publish 성공·실패 | Fake Repository override | 외부 결과를 제어해야 함 |
| `loading → data → error` 순서 | `AsyncNotifier` + Fake | 비동기 상태 전이가 핵심 |
| Provider 수명과 정리 | `ProviderContainer.test` + listener | dispose 시점을 확인해야 함 |

실제 프로젝트에서 처음부터 모든 테스트를 `overrideWithBuild`로 바꾸지는 않았다. 초기 상태가 복잡한 대시보드나 로그인 이후 화면처럼 “특정 상태에서 사용자 액션”을 빠르게 재현할 때만 사용했다. 반대로 AWS IoT 연결, MQTT ACK 타임아웃, 재연결 백오프는 Repository Fake를 둬야 테스트가 현실을 반영한다.

## 주의할 점

`overrideWithBuild`가 Repository 의존성까지 자동으로 없애 주는 건 아니다. `build()` 안에서 `ref.watch(mqttRepositoryProvider)`를 호출하면 해당 Repository는 여전히 생성될 수 있다. 테스트에서 네트워크 연결이 시작됐다면 Repository Provider도 별도로 override해야 한다.

또 하나는 상태 객체의 가변성이다. `build()`에서 같은 가변 객체를 돌려주면 테스트 사이에 값이 남을 수 있다. 매번 새 `BoilerState`를 반환하고, 각 테스트에서 새 `ProviderContainer.test()`를 만드는 기준을 지키는 편이 안전하다.

솔직하게 정리하면 `overrideWithBuild`는 Mock을 없애는 기능이 아니라 Mock의 범위를 줄이는 기능이다. Flutter 스마트홈 화면의 초기 조건은 간단히 주입하고, 사용자가 누른 명령과 상태 전이는 원래 Notifier 코드로 검증한다. MQTT 통신 품질까지 확인해야 하는 테스트에서는 Fake Repository를 함께 남겨야 한다.

