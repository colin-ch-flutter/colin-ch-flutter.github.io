---
layout: post
title: "Flutter Dart sealed class 패턴 매칭 - MQTT 상태를 안전하게 모델링하는 방법"
description: "Flutter Dart sealed class와 패턴 매칭으로 MQTT 연결 상태를 표현하고, 누락된 상태를 컴파일 타임에 잡아 IoT 화면과 테스트를 단순하게 만드는 방법을 정리했다."
date: 2026-09-02
tags: [Flutter, Dart, MQTT, IoT, 테스트, 상태관리]
comments: true
share: true
---

![Flutter Dart sealed class로 MQTT 상태를 모델링하는 코드 화면](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&q=80)

_상태를 문자열과 nullable 필드로 흩어 놓으면 MQTT 재연결 화면에서 어떤 조합이 가능한지 알기 어렵다._

Flutter IoT 앱에서 MQTT 상태를 `sealed class`로 만들고 패턴 매칭으로 그리면, 연결·재연결·오류 화면의 누락을 컴파일 단계에서 줄일 수 있다. 처음엔 enum 하나면 충분하다고 생각했는데 아니었다. `reconnecting`에 남은 시간, `connected`에 마지막 수신 시각처럼 상태마다 필요한 데이터가 달랐다.

## enum과 nullable 필드가 만든 문제

기존 코드는 대략 이런 모양이었다.

```dart
enum MqttStatus { disconnected, connecting, connected, error }

class MqttUiState {
  final MqttStatus status;
  final String? message;
  final int? retryCount;
  // ...
}
```

`connected`인데 `message`가 남아 있거나, `error`인데 메시지가 null인 조합도 막을 수 없다. Controller가 상태를 바꿀 때마다 필드 초기화 순서를 신경 써야 했고, 실제로 재연결 중 이전 오류 문구가 카드에 남는 버그가 있었다.

## sealed class로 상태의 경계를 만든다

상태별 데이터를 생성자에 묶으면 잘못된 조합 자체가 줄어든다. 아래 코드는 MQTT 연결 상태를 화면에서 사용하는 최소 모델이다.

```dart
sealed class MqttState {
  const MqttState();
}

final class MqttDisconnected extends MqttState {
  const MqttDisconnected();
}

final class MqttConnecting extends MqttState {
  final int attempt;
  const MqttConnecting({required this.attempt});
}

final class MqttConnected extends MqttState {
  final DateTime connectedAt;
  const MqttConnected({required this.connectedAt});
}

final class MqttError extends MqttState {
  final String message;
  const MqttError(this.message);
}
```

`sealed`는 이 라이브러리 안에서 파생 타입을 통제한다. 그래서 `switch`가 모든 상태를 처리하는지 Dart 분석기가 확인할 수 있다. 새 상태인 `MqttOffline`을 추가했는데 화면 switch에서 처리하지 않으면 경고가 바로 보인다.

## switch expression으로 UI를 단순하게 만든다

상태에 따라 카드 문구와 색상을 함께 결정해야 하므로, `if`를 길게 늘어뜨리기보다 switch expression을 사용했다.

```dart
({String label, Color color}) viewData(MqttState state) => switch (state) {
      MqttDisconnected() => (label: '연결 안 됨', color: Colors.grey),
      MqttConnecting(:final attempt) => (
          label: '재연결 중 ($attempt회)',
          color: Colors.orange,
        ),
      MqttConnected(:final connectedAt) => (
          label: '연결됨 · ${connectedAt.hour}:${connectedAt.minute}',
          color: Colors.green,
        ),
      MqttError(:final message) => (
          label: '오류: $message',
          color: Colors.red,
        ),
    };
```

여기서 `MqttConnecting(:final attempt)`처럼 필요한 값만 꺼낼 수 있다. UI 위젯은 각 상태의 내부 필드를 알 필요 없이 `viewData` 결과만 그리면 된다. 상태 모델과 표현 로직을 분리한 덕분에 Widget Test도 네트워크 연결 없이 같은 결과를 검증할 수 있다.

| 방식 | 상태별 데이터 | 누락 상태 감지 | 잘 맞는 경우 |
|---|---|---|---|
| enum + nullable | 필드 조합으로 관리 | 약함 | 상태가 단순할 때 |
| sealed class | 타입별 생성자에 고정 | 강함 | MQTT·BLE·인증 상태 |
| 문자열 상태 | 외부 규칙에 의존 | 없음 | 임시 로그·서버 원문 |

## 테스트에서 확인할 지점

패턴 매칭 함수는 순수 함수로 뽑았기 때문에 테스트가 짧다.

```dart
test('재연결 횟수를 상태 문구에 표시한다', () {
  final data = viewData(const MqttConnecting(attempt: 3));

  expect(data.label, '재연결 중 (3회)');
  expect(data.color, Colors.orange);
});
```

주의할 점도 있다. `sealed class`가 모든 비즈니스 상태를 자동으로 관리해 주는 건 아니다. MQTT 연결 객체에서 상태를 발행하는 Repository가 여전히 `MqttError`와 `MqttConnecting`을 올바른 순서로 내보내야 한다. 또 SDK 하위 호환이 필요한 프로젝트라면 사용하는 Dart 버전과 분석기 설정을 확인해야 한다. 언어 기능만 도입하고 프로젝트의 최소 SDK를 올리지 않으면 CI에서만 실패할 수 있다.

솔직하게 정리하면, 상태가 네 개 이하일 때는 enum이 더 빠를 수 있다. 하지만 상태마다 데이터가 달라지고 화면·Controller·테스트가 같은 상태를 공유하기 시작했다면 sealed class가 비용을 갚는다. 특히 MQTT 재연결처럼 오류와 진행 정보가 함께 움직이는 화면에서는 nullable 필드를 줄이는 효과가 바로 보인다.

짧게 요점만 남기면:

- 상태별 데이터가 다르면 enum의 nullable 필드부터 의심한다.
- `sealed class`와 switch expression으로 상태 누락을 분석기에 맡긴다.
- UI에 패턴 매칭을 흩뿌리지 말고 순수 변환 함수로 분리한다.
- 새 상태를 추가한 뒤 Widget Test와 CI의 Dart SDK 버전을 함께 확인한다.
