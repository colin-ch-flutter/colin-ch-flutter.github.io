---
layout: post
title: "Flutter integration_test MQTT 멱등성 검증 - IoT 명령 중복 실행 막기"
description: "Flutter integration_test에서 MQTT QoS 1 재전달과 네트워크 재시도로 같은 IoT 명령이 두 번 실행되는 문제를 재현하고, commandId 멱등성 검증으로 스마트홈 제어를 안정화하는 방법을 정리했다."
date: 2026-08-23
tags: [Flutter, MQTT, mqtt5_client, IoT, 스마트홈, CI/CD]
comments: true
share: true
---

![스마트홈 MQTT 명령 멱등성 테스트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter integration_test에서 MQTT 명령의 멱등성을 검증하지 않으면, 네트워크가 잠깐 끊긴 뒤 보일러나 조명이 두 번 바뀔 수 있다. QoS 1은 전달을 보장하지만 중복 전달이 가능하기 때문이다. `commandId`로 이미 처리한 명령을 건너뛰고, 같은 이벤트를 두 번 주입해 실행 횟수가 1인지 확인했다.

## 재연결 테스트만으로는 부족했다

처음에는 MQTT 연결이 끊겼다가 다시 연결되는지만 확인했다. 실제 현장에서는 재연결 직후 서버가 이전 명령을 다시 보내거나, 앱이 타임아웃으로 같은 명령을 재발행했다. 최종 상태만 보면 같아서 테스트 로그를 보지 않고는 발견하기 어렵다.

| 상황 | 기대 동작 | 놓치면 생기는 문제 |
|---|---|---|
| 같은 `commandId` 재전달 | 한 번만 실행 | 릴레이 이중 토글 |
| 다른 ID, 같은 값 | 각각 실행 | 정상 명령 누락 |
| 처리 중인 ID 재수신 | 완료 전에도 무시 | 중복 호출 |

## 처리 전에 ID를 예약한다

실제 디바이스 대신 호출 횟수를 기록하는 Fake Repository를 주입했다. 핵심은 상태를 바꾼 뒤 ID를 저장하지 않고, 처리 시작 전에 예약하는 순서다.

```dart
class CommandDeduplicator {
  final Set<String> _handled = <String>{};

  bool reserve(String commandId) {
    if (_handled.contains(commandId)) return false;
    _handled.add(commandId);
    return true;
  }
}

Future<void> onMqttMessage(MqttCommand command) async {
  if (!_deduplicator.reserve(command.commandId)) return;
  await _deviceRepository.apply(command.target, command.value);
}
```

`apply()` 실패 후 재시도를 허용해야 한다면 예약과 완료를 분리한다. 반대로 모터나 릴레이처럼 실행 자체가 위험한 명령은 실패해도 같은 ID를 막는 편이 안전할 수 있다. 이 정책은 문서가 아니라 코드와 테스트에 남겨야 한다.

## 같은 메시지를 두 번 주입한다

실제 브로커 대신 Fake MQTT 서비스가 수신 이벤트를 주입한다. 같은 `commandId`를 두 번 보내 QoS 1 재전달을 작게 재현한다.

```dart
testWidgets('같은 MQTT commandId는 한 번만 실행된다', (tester) async {
  final fakeMqtt = FakeMqttService();
  final fakeDevice = FakeDeviceRepository();

  await pumpSmartHomeApp(tester, mqtt: fakeMqtt, device: fakeDevice);

  final message = MqttCommand(
    commandId: 'cmd-100', target: 'boiler', value: 'on',
  );
  fakeMqtt.emit(message);
  fakeMqtt.emit(message);
  await tester.pumpAndSettle();

  expect(fakeDevice.applyCount('boiler'), 1);
});
```

`pumpAndSettle()`만 믿으면 비동기 처리보다 테스트가 먼저 끝날 수 있다. Fake Repository에 `Completer`를 두고 완료 시점을 기다리거나, 제한 시간 안에 실행 횟수가 1이 되는지 확인해야 한다.

## 저장 범위는 명령 성격에 따라 정한다

메모리의 `Set`만으로는 앱 재시작 후 중복을 막지 못한다. 재생 가능성이 긴 기기라면 Realm에 최근 처리 ID와 만료 시간을 저장한다. ID를 무제한으로 쌓지 않도록 보관 기간도 정해야 한다.

MQTT 재연결 테스트는 연결 상태만으로 끝나지 않는다. 같은 메시지를 두 번 주입했을 때 실제 장치 Repository 호출이 한 번인지까지 검증해야 한다. “최종 상태가 같다”와 “명령이 한 번 실행됐다”는 전혀 다른 주장이다.
