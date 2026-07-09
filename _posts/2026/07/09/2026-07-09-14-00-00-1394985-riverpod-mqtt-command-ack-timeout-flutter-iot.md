---
layout: post
title: "Flutter IoT Riverpod MQTT ACK - 보일러 명령 응답 대기와 타임아웃 나누기"
description: "Flutter IoT 앱에서 Riverpod으로 Flutter 스마트홈 MQTT 명령의 pending, ack, timeout 상태를 나눠 보일러 제어 실패를 덜 헷갈리게 만든 방법이다."
date: 2026-07-09
tags: [Flutter, Riverpod, MQTT, mqtt5_client, IoT]
comments: true
share: true
---

![Flutter IoT Riverpod MQTT ACK](https://images.unsplash.com/photo-1558346490-a72e53ae2d4f?w=800&q=80)
이 그림에서는 앱 화면, MQTT 브로커, 실제 기기 사이에 응답 지연이 생기는 구간을 봐야 한다.

Flutter IoT 앱에서 Flutter 스마트홈 보일러 명령은 MQTT publish 성공만으로 끝내면 안 됐다. Riverpod으로 상태를 옮긴 뒤에도 `mqtt5_client`가 publish를 반환했다고 바로 성공 처리하면, 실제 기기가 명령을 적용하지 못한 상황을 놓친다. 전원 토글과 목표 온도 변경은 `pending`, `ack`, `timeout`을 따로 봐야 덜 헷갈렸다.

처음엔 publish가 실패하면 에러, 성공하면 완료라고 생각했다. 로컬 Wi-Fi에서는 그럭저럭 맞았다. 그런데 LTE에서 AWS IoT Core를 거치면 reported 메시지가 1~2초 늦게 오고, 기기 전원이 약한 상태에서는 publish는 성공했는데 보일러는 그대로인 경우가 있었다.

## publish와 ack를 분리한다

명령 상태는 화면 상태와 다르게 짧고 사건 중심이다. 그래서 읽기 Provider에 섞지 않고 Command Notifier에 따로 뒀다.

| 상태 | 의미 | 화면 처리 |
|---|---|---|
| `idle` | 보낼 명령 없음 | 버튼 활성화 |
| `pending` | publish 후 기기 응답 대기 | 버튼 잠금, 작은 로딩 |
| `acked` | reported 값이 기대값과 일치 | 성공 표시 후 해제 |
| `timeout` | 정해진 시간 안에 반영 안 됨 | 재시도 안내 |

`publish()`는 브로커까지 보낸 결과이고, ack는 기기 상태 스트림에서 기대값을 확인한 결과다.

이 코드는 전원 명령의 대기값을 잡고, reported 값이 맞으면 pending을 닫는 흐름이다.
```dart
class BoilerCommandNotifier extends Notifier<BoilerCommandState> {
  Timer? _timeoutTimer;

  @override
  BoilerCommandState build() {
    ref.onDispose(() => _timeoutTimer?.cancel());
    return const BoilerCommandState();
  }

  Future<void> setPower(String deviceId, bool power) async {
    if (state.isPending) return;

    state = BoilerCommandState(pendingPower: power);
    _timeoutTimer?.cancel();
    _timeoutTimer = Timer(const Duration(seconds: 4), () {
      if (state.pendingPower == power) {
        state = const BoilerCommandState(errorMessage: '기기 응답이 늦습니다');
      }
    });

    await ref.read(boilerRepositoryProvider).publishPower(deviceId, power);
  }

  void onReportedPower(bool reportedPower) {
    if (state.pendingPower == reportedPower) {
      _timeoutTimer?.cancel();
      state = const BoilerCommandState();
    }
  }
}
```

처음엔 타임아웃을 1초로 잡았다. 앱은 빠릿해 보였지만 실제 환경에서는 너무 짧았다. LTE와 공유기 재연결 상황을 섞어보니 3~5초가 더 현실적이었다.

## reported 값만 믿는다

여기서 헷갈렸던 건 retained 메시지였다. 앱이 재실행되자마자 오래된 reported 값을 받고 pending을 해제한 적이 있다. 그래서 명령을 보낸 시각 이후의 메시지만 ack 후보로 봤다. 서버에서 commandId를 내려줄 수 있으면 그게 더 낫다.

주의할 점도 있다. 낙관적 UI를 완전히 금지할 필요는 없다. 다만 보일러 전원처럼 실제 상태 착각이 위험한 값은 “켜진 것처럼” 바꾸기보다 “켜는 중”으로 보여주는 편이 안전했다.

짧게 남기면 이렇다. Flutter IoT Riverpod 구조에서 MQTT 명령은 publish 성공, 기기 ack, timeout을 분리해야 한다. 화면은 pending을 보여주고, 실제 성공은 reported 값으로 닫는다. 그래야 Flutter 스마트홈 보일러 제어에서 MQTT 지연을 사용자에게 덜 거짓말하게 된다.
