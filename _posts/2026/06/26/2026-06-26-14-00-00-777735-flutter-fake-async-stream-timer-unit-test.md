---
layout: post
title: "Flutter fakeAsync 비동기 테스트 - Timer·Stream 타이밍 이슈 완전 정복"
description: "fakeAsync로 Flutter 비동기 테스트에서 Timer 타임아웃과 Stream 이벤트를 실시간 대기 없이 즉시 검증하는 방법을 MQTT 재연결·BLE 스캔 예제로 정리했다."
date: 2026-06-26
tags: [Flutter, Dart, MQTT, BLE, IoT]
comments: true
share: true
---

![Flutter fakeAsync 비동기 테스트 타이머 제어](https://images.unsplash.com/photo-1501139083538-0139583c060f?w=800&q=80)

MQTT 재연결 타이머를 테스트하려고 `await Future.delayed(Duration(seconds: 5))`를 호출했다가 테스트 스위트 전체가 30초 넘게 걸렸다. BLE 스캔 타임아웃 로직도 마찬가지였다. 실제 시간이 흐르길 기다리는 테스트는 느리고 CI에서 불규칙하게 실패한다. `fakeAsync`를 쓰면 5초짜리 타이머를 코드 한 줄로 즉시 진행시킬 수 있다.

## 왜 `await Future.delayed()`가 테스트에서 문제가 되는가

IoT 앱에서 타이머가 필요한 상황은 많다.

- MQTT 연결 끊김 후 5초 대기 → 재연결 시도
- BLE 스캔 시작 10초 후 자동 중지
- 보일러 제어 명령 후 3초 이내 응답 없으면 타임아웃 처리
- 센서 데이터 Stream에서 2초 이상 값이 안 오면 오프라인 표시

테스트에서 실제 `Timer`를 기다리면 세 가지 문제가 생긴다.

1. **느리다** — 타이머 개수만큼 시간이 실제로 흐른다
2. **불안정하다** — CI 머신 부하에 따라 타이밍이 달라져서 간헐적 실패
3. **격리가 안 된다** — 테스트 간 타이머가 겹치면 이전 테스트 타이머가 다음 테스트에 영향

## fakeAsync 기본 구조

```dart
import 'package:fake_async/fake_async.dart';
import 'package:test/test.dart';

test('5초 타이머가 지나면 재연결 시도', () {
  fakeAsync((async) {
    // 여기 안에서 실행되는 모든 Timer, Future.delayed는 가짜 시계를 쓴다
    var reconnectCalled = false;
    Timer(Duration(seconds: 5), () => reconnectCalled = true);

    // 실제로는 즉시 실행, 가짜 시계를 5초 앞으로 돌린다
    async.elapse(Duration(seconds: 5));

    expect(reconnectCalled, isTrue);
  });
});
```

`fakeAsync` 콜백 안에서는 `Timer`, `Future.delayed`, `Stream.periodic`이 모두 가짜 시계에 묶인다. `async.elapse()`로 원하는 만큼 시간을 즉시 앞으로 돌릴 수 있다.

### 주요 메서드

| 메서드 | 역할 |
|--------|------|
| `async.elapse(duration)` | 지정한 시간만큼 가짜 시계 진행 + 그 사이 타이머 실행 |
| `async.flushTimers()` | 큐에 남은 모든 타이머 즉시 실행 |
| `async.flushMicrotasks()` | 마이크로태스크 큐 전부 비우기 |
| `async.pendingTimers` | 아직 실행 안 된 타이머 개수 확인 |

## MQTT 재연결 타이머 테스트

[이전 글에서 MqttService를 Fake로 격리하는 방법을 다뤘다](/2026/06/24/mqtt5-client-unit-test-fake-service-isolation/). 그걸 이어받아 재연결 타이머 로직을 테스트한다.

실제 코드에서 `MqttController`는 연결이 끊기면 5초 뒤 재연결을 시도한다.

```dart
// mqtt_controller.dart
class MqttController extends GetxController {
  final MqttService _mqtt;
  Timer? _reconnectTimer;

  void onDisconnected() {
    _reconnectTimer?.cancel();
    _reconnectTimer = Timer(
      const Duration(seconds: 5),
      () => _mqtt.connect(),
    );
  }
}
```

이걸 테스트할 때 `Future.delayed`로 기다리면 5초가 실제로 흐른다. `fakeAsync`를 쓰면 다르다.

```dart
test('연결 끊김 후 5초 뒤 재연결 시도', () {
  fakeAsync((async) {
    final fakeMqtt = FakeMqttService();
    final controller = MqttController(mqtt: fakeMqtt);

    controller.onDisconnected();

    // 4초 후에는 아직 재연결 안 됨
    async.elapse(Duration(seconds: 4));
    expect(fakeMqtt.connectCallCount, 0);

    // 5초 지나면 재연결
    async.elapse(Duration(seconds: 1));
    expect(fakeMqtt.connectCallCount, 1);
  });
});
```

4초 구간 검증이 포인트다. "5초 후에 된다"는 것만 확인하면 타이머가 즉시 실행되는 버그를 못 잡는다. 경계값 전후를 둘 다 검증해야 한다.

## BLE 스캔 타임아웃 테스트

BLE 스캔은 10초 후 자동으로 멈추게 만들었다. 실기기가 없으면 테스트할 수 없는 코드였는데 `fakeAsync` + Fake BLE 서비스 조합으로 해결했다.

```dart
// ble_controller.dart
class BleController extends GetxController {
  final BleService _ble;
  final isScanning = false.obs;
  Timer? _scanTimer;

  void startScan() {
    isScanning.value = true;
    _ble.startScan();
    _scanTimer = Timer(const Duration(seconds: 10), stopScan);
  }

  void stopScan() {
    _scanTimer?.cancel();
    isScanning.value = false;
    _ble.stopScan();
  }
}
```

```dart
test('스캔 시작 10초 후 자동 중지', () {
  fakeAsync((async) {
    final fakeBle = FakeBleService();
    final controller = BleController(ble: fakeBle);

    controller.startScan();
    expect(controller.isScanning.value, isTrue);

    async.elapse(Duration(seconds: 9));
    expect(controller.isScanning.value, isTrue);   // 아직 진행 중

    async.elapse(Duration(seconds: 1));
    expect(controller.isScanning.value, isFalse);  // 자동 중지
    expect(fakeBle.stopScanCalled, isTrue);
  });
});
```

## Stream 테스트 - expectLater 패턴

센서 데이터처럼 Stream으로 흘러오는 값은 `expectLater`를 쓴다. 근데 여기서 함정이 있다.

`expectLater`는 Future를 반환하는데, `fakeAsync` 안에서 `await`하면 **데드락**이 걸린다. `fakeAsync` 블록이 동기적으로 실행되기 때문이다.

```dart
// 이렇게 하면 데드락 - fakeAsync 안에서 await 금지
test('잘못된 패턴', () {
  fakeAsync((async) async {  // ← async 쓰면 문제
    final stream = controller.temperatureStream;
    await expectLater(stream, emits(22.5));  // ← 여기서 멈춤
  });
});
```

```dart
// 올바른 패턴 - await 없이 expectLater 먼저, 그 다음 이벤트 발생
test('온도 Stream 값 검증', () {
  fakeAsync((async) {
    final controller = SensorController(sensor: fakeSensor);

    // 1. expectLater를 await 없이 먼저 등록
    final expectation = expectLater(
      controller.temperatureStream,
      emitsInOrder([22.5, 23.0]),
    );

    // 2. 이벤트 발생시키기
    fakeSensor.emitTemperature(22.5);
    fakeSensor.emitTemperature(23.0);

    // 3. 마이크로태스크 비워서 Stream 이벤트 전파
    async.flushMicrotasks();

    // 4. expectation이 완료됐는지 확인
    // (비동기 완료를 기다리지 않아도 flushMicrotasks 후에는 이미 검증됨)
  });
});
```

처음에 이 순서를 거꾸로 했다가 반나절을 날렸다. `expectLater`는 코드 under test보다 **반드시 먼저** 등록해야 한다.

## 삽질: flushTimers() 무한 루프

MQTT `keepAlive` 핑을 주기적으로 보내는 코드를 테스트하다가 `flushTimers()`가 무한 루프에 빠졌다.

```dart
// 문제 코드: 타이머가 자기 자신을 다시 등록
void startKeepAlive() {
  Timer(Duration(seconds: 30), () {
    _sendPing();
    startKeepAlive();  // ← 타이머 안에서 새 타이머 등록 → flushTimers 무한 루프
  });
}
```

`flushTimers()`는 현재 등록된 타이머를 모두 실행하는데, 실행 중에 새 타이머가 등록되면 그것도 실행한다. 반복 타이머는 반드시 `elapse`로 딱 필요한 구간만 진행해야 한다.

```dart
test('keepAlive 핑 30초마다 전송', () {
  fakeAsync((async) {
    controller.startKeepAlive();

    async.elapse(Duration(seconds: 30));
    expect(fakeMqtt.pingCount, 1);

    async.elapse(Duration(seconds: 30));
    expect(fakeMqtt.pingCount, 2);
  });
});
```

## fakeAsync vs WidgetTester pump

Widget Test에서는 `pump(duration)`이 비슷한 역할을 한다. 차이는 이렇다.

| | `fakeAsync` | `pump(duration)` |
|--|-------------|-----------------|
| 용도 | Unit Test | Widget Test |
| 가짜 시계 | `fake_async` 패키지 | Flutter 테스트 프레임워크 내장 |
| UI 갱신 | 없음 | 프레임 재빌드 포함 |

Unit Test에서는 `fakeAsync`, Widget Test에서는 `tester.pump()`를 쓴다. 섞어 쓰면 예상 못 한 동작이 나온다.

---

다음 편은 테스트 커버리지 목표 설정과 유지 전략 — 어느 레이어를 몇 퍼센트까지 커버할지, 커버리지 숫자가 높아도 테스트 품질이 낮은 경우를 다룬다.

**참고 링크**
- [FakeAsync class - Dart API](https://api.flutter.dev/flutter/package-fake_async_fake_async/FakeAsync-class.html)
- [Mastering Time in Flutter Tests - Medium](https://cdmunoz.medium.com/mastering-time-in-flutter-tests-how-fakeasync-eliminates-flaky-tests-and-delivers-precision-0e0d909e6b88)
- [How to Write Tests using Stream Matchers - Code with Andrea](https://codewithandrea.com/articles/async-tests-streams-flutter/)
