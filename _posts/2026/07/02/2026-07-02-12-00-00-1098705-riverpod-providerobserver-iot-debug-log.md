---
layout: post
title: "Flutter IoT Riverpod ProviderObserver - BLE와 MQTT 상태 변경 로그 남기기"
description: "Flutter IoT 앱에서 Riverpod ProviderObserver로 Flutter BLE 재연결과 MQTT 연결 끊김 상태 변화를 추적하고 디버깅 로그를 정리한 방법이다."
date: 2026-07-02
tags: [Flutter, Dart, BLE, MQTT, IoT]
comments: true
share: true
---
![Flutter IoT Riverpod ProviderObserver](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

Flutter IoT 앱에서 Riverpod `ProviderObserver`를 붙이면 Flutter BLE 재연결과 MQTT 연결 끊김 같은 상태 변화를 한곳에서 볼 수 있다. Flutter 스마트홈 앱을 운영하다 보면 화면에서 보이는 에러보다 “언제 상태가 바뀌었는지”가 더 중요할 때가 많다.

처음엔 각 Controller나 Notifier 안에 `debugPrint()`를 넣었다. BLE 스캔 시작, 연결 성공, MQTT subscribe 완료, 재연결 실패 지점마다 로그를 박았다. 해보니 됐지만 오래 못 갔다. 로그 포맷이 제각각이었고, Provider를 Riverpod으로 옮긴 뒤에는 상태가 어디서 바뀌었는지 따라가기 더 어려웠다. 특히 기기 상세 화면을 나갔다 들어오면 Provider가 dispose됐다가 다시 생성되는데, 이 흐름을 놓치면 “왜 연결이 다시 끊긴 것처럼 보이지?” 같은 질문만 남는다.

## ProviderObserver는 상태 변경 블랙박스다

`ProviderObserver`는 Provider 값이 추가, 변경, 삭제될 때 호출된다. 앱 전체에 하나 붙여두면 특정 화면 코드에 로그를 섞지 않아도 된다. 개발 빌드에서만 켜고, 운영 빌드에서는 Crashlytics나 내부 로그 수집기로 보내는 식으로 분리하기도 쉽다.

ProviderScope에 Observer를 넣는 코드는 단순하다.

```dart
void main() {
  runApp(
    ProviderScope(
      observers: [
        IotProviderObserver(),
      ],
      child: const App(),
    ),
  );
}
```

실제로 쓴 Observer는 모든 Provider를 찍지 않았다. 화면 테마, 다국어, 폼 입력값까지 전부 남기면 필요한 로그가 묻힌다. BLE와 MQTT처럼 장애 분석에 필요한 Provider만 필터링했다.

```dart
class IotProviderObserver extends ProviderObserver {
  @override
  void didUpdateProvider(
    ProviderBase<Object?> provider,
    Object? previousValue,
    Object? newValue,
    ProviderContainer container,
  ) {
    final name = provider.name;
    if (name == null) return;

    final isIotState =
        name.startsWith('ble') || name.startsWith('mqtt');
    if (!isIotState) return;

    debugPrint(
      '[riverpod] $name: $previousValue -> $newValue',
    );
  }
}
```

여기서 핵심은 Provider에 이름을 붙이는 것이다. 이름이 없으면 로그에 `AutoDisposeNotifierProviderImpl` 같은 내부 타입만 남아서 별 도움이 안 된다.

```dart
final mqttConnectionProvider =
    NotifierProvider.family<MqttConnectionNotifier,
        AsyncValue<MqttStatus>, String>(
  MqttConnectionNotifier.new,
  name: 'mqttConnectionProvider',
);
```

## 로그는 적게 남겨야 읽힌다

솔직하게 정리해보면 Observer를 붙인 첫날엔 로그가 너무 많았다. BLE 스캔 결과 Stream이 계속 흘러오면서 콘솔이 밀렸고, 정작 보고 싶던 MQTT 재연결 실패는 지나가 버렸다. 그래서 값 전체보다 상태 이름, deviceId, 에러 코드 정도만 남기는 쪽으로 줄였다.

BLE 재연결 문제는 특히 이 방식이 잘 맞았다. iOS 실기기에서 앱을 백그라운드로 보낸 뒤 다시 열었을 때 `disconnected -> scanning -> connecting -> failed` 순서가 찍혔다. 예전에는 마지막 에러만 보고 권한 문제라고 의심했는데, 실제로는 재연결 타이머가 두 번 돌면서 같은 기기에 중복 connect를 시도한 게 원인이었다.

주의할 점도 있다. 토큰, Wi-Fi SSID, 사용자 이메일 같은 값은 Observer에서 그대로 찍으면 안 된다. Flutter IoT 앱은 기기 식별자와 네트워크 정보가 로그에 섞이기 쉽다. 나는 Provider 값을 그대로 출력하지 않고 `toLogString()` 같은 메서드로 노출해도 되는 필드만 남기는 방식을 썼다.

짧게 정리하면 이렇다. `ProviderObserver`는 Riverpod 상태를 전역에서 훔쳐보는 도구가 아니라, BLE와 MQTT 장애 흐름을 재현하기 위한 디버깅 장치다. 모든 Provider를 찍지 말고, 이름을 붙이고, 민감한 값은 제거하고, 연결 상태 전이만 남기면 실제 장애 분석 시간이 꽤 줄어든다.
