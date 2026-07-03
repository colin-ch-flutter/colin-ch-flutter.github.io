---
layout: post
title: "Flutter IoT Riverpod offline queue - MQTT 끊김 중 명령을 안전하게 보관하기"
description: "Flutter IoT 앱에서 Riverpod과 Repository로 MQTT 오프라인 명령 큐를 분리해 화면 이동과 재연결 중 중복 publish를 막는 방법이다."
date: 2026-07-03
tags: [Flutter, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---
![Flutter IoT Riverpod offline queue](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

이 그림에서는 네트워크가 끊긴 동안에도 앱 안의 사용자 의도는 사라지지 않아야 한다는 점을 봐야 한다.

Flutter IoT 앱에서 MQTT가 끊겼을 때 버튼을 막기만 하면 사용자는 답답하고, 끊긴 상태로 publish를 날리면 명령이 사라진다. 내가 확인한 기준으론 Riverpod 화면 상태와 오프라인 명령 큐를 분리하는 편이 가장 덜 흔들렸다. 화면은 마지막 사용자 의도만 보여주고, 실제 전송 대기는 Repository 아래에서 관리한다.

처음엔 `isConnected == false`면 전원 버튼을 비활성화했다. 안전해 보였지만 LTE가 잠깐 흔들리는 지하 주차장에서 문제가 생겼다. 사용자는 보일러를 끄고 싶은데 앱은 3초 동안 아무것도 못 하게 했다. 반대로 버튼을 열어두면 재연결 직후 같은 명령이 두 번 전송됐다.

| 상황 | 화면 처리 | 큐 처리 |
|---|---|---|
| MQTT 연결됨 | 즉시 전송 결과 표시 | 큐 비움 |
| 연결 끊김 | 대기 중 표시 | 명령 1개 저장 |
| 같은 기기 연속 탭 | 마지막 의도만 표시 | 이전 명령 교체 |
| 재연결 성공 | 동기화 중 표시 | 순서대로 flush |

핵심은 큐를 Widget에 두지 않는 것이다.

```dart
class PendingCommand {
  const PendingCommand({
    required this.deviceId,
    required this.topic,
    required this.payload,
    required this.createdAt,
  });

  final String deviceId;
  final String topic;
  final String payload;
  final DateTime createdAt;
}
```

기기별 마지막 명령만 남기면 보일러 전원을 켰다가 바로 끄는 상황에서도 오래된 명령이 뒤늦게 나가지 않는다.

```dart
class MqttCommandQueue {
  final _items = <String, PendingCommand>{};

  void enqueue(PendingCommand command) {
    _items[command.deviceId] = command;
  }

  List<PendingCommand> drain() {
    final values = _items.values.toList()
      ..sort((a, b) => a.createdAt.compareTo(b.createdAt));
    _items.clear();
    return values;
  }
}
```

Riverpod에서는 큐 상태를 화면에서 직접 고치지 않고, 연결 상태 변화에 맞춰 flush만 호출했다.

```dart
ref.listen(mqttSessionProvider, (previous, next) {
  if (previous?.connected == false && next.connected) {
    ref.read(commandRepositoryProvider).flushPendingCommands();
  }
});
```

여기서 헷갈렸던 건 모든 명령을 보관해야 한다는 생각이었다. 온도 22도, 23도, 24도를 1초 안에 누른 경우라면 세 개를 모두 보낼 필요가 없다. 마지막 값 24도만 의미가 있다. 반대로 문 열림 알림처럼 사건 로그는 덮어쓰면 안 된다. 명령인지 이벤트인지 기준을 나눠야 한다.

주의할 점은 큐 보관 시간이다. 10분 전에 눌렀던 보일러 켜기 명령이 재연결 후 갑자기 실행되면 더 위험하다. 나는 제어 명령은 30초, 화면 새로고침은 10초처럼 짧게 잡고 시간이 지난 명령은 버리는 쪽이 낫다고 봤다.

짧게 정리하면 이렇다.

- Flutter IoT 앱에서 오프라인 명령 큐는 화면 상태가 아니라 Repository 책임이다.
- 같은 deviceId의 제어 명령은 마지막 사용자 의도만 남기는 편이 안전하다.
- MQTT 재연결 시 `ref.listen`으로 flush 타이밍만 연결한다.
- 오래된 명령은 실행보다 폐기가 더 나은 경우가 많다.
