---
layout: post
title: "Flutter IoT Riverpod ref.listen - 스마트홈 부수효과 분리하기"
description: "Flutter IoT 앱에서 Riverpod ref.listen으로 Flutter 스마트홈 MQTT 연결 끊김과 Flutter BLE 재연결 알림 같은 부수효과를 분리한 방법을 정리했다."
date: 2026-07-02
tags: [Flutter, Dart, BLE, MQTT, IoT]
comments: true
share: true
---
![Flutter IoT Riverpod ref.listen](https://images.unsplash.com/photo-1517420704952-d9f39e95b43e?w=800&q=80)

Flutter IoT 앱에서 Riverpod으로 상태를 옮길 때 `ref.listen`은 화면 갱신보다 부수효과를 분리하는 데 더 유용했다. Flutter 스마트홈 화면에서 MQTT 연결 끊김, Flutter BLE 재연결 실패, 토스트 알림, 로그 전송까지 `build()` 안에 섞으면 상태관리가 다시 지저분해진다.

처음엔 `ref.watch()`만 쓰면 된다고 생각했다. MQTT 상태 Provider를 watch하고, 값이 `disconnected`면 SnackBar를 띄우면 된다고 봤다. 해보니 아니었다. 위젯이 리빌드될 때마다 같은 SnackBar가 반복됐고, BLE 스캔 화면에서는 뒤로 갔다가 다시 들어오는 순간 이전 에러 알림이 또 떴다. GetX Controller가 비대해지는 문제를 피하려고 Riverpod을 넣었는데, 이번엔 Widget에 부수효과가 붙기 시작했다.

## watch는 화면 값, listen은 사건 처리

`ref.watch()`는 UI가 그릴 값을 읽을 때 쓴다. 온도, 전원 상태, 연결 아이콘처럼 화면에 계속 보여야 하는 값이 여기에 맞다. 반대로 `ref.listen()`은 값이 바뀌는 순간 한 번 실행할 작업에 맞다. 예를 들면 MQTT 연결이 `connected`에서 `disconnected`로 바뀌었을 때 로그를 남기거나, BLE 재연결 실패를 사용자에게 알리는 식이다.

MQTT 연결 끊김을 화면 밖 부수효과로 분리하면 이렇게 된다.

```dart
class DeviceDetailPage extends ConsumerWidget {
  const DeviceDetailPage({super.key, required this.deviceId});

  final String deviceId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.listen(mqttStatusProvider(deviceId), (previous, next) {
      if (previous?.valueOrNull == MqttStatus.connected &&
          next.valueOrNull == MqttStatus.disconnected) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('기기 연결이 끊겼다')),
        );
      }
    });

    final status = ref.watch(deviceStatusProvider(deviceId));
    return DeviceStatusView(status: status);
  }
}
```

핵심은 UI 상태와 사건 처리를 같은 Provider에서 읽더라도 역할을 나누는 것이다. `watch`는 `DeviceStatusView`를 다시 그리는 데만 쓰고, `listen`은 연결 상태 전이를 감지하는 데만 쓴다.

## 실제로 막힌 부분

가장 많이 실수한 건 `next`만 보는 코드였다. `next`가 disconnected인지 확인하면 페이지 진입 시점에 이미 끊긴 기기도 알림을 띄운다. 스마트홈 앱에서는 사용자가 상세 화면에 들어올 때 오프라인 기기가 꽤 많다. 이 경우는 장애 알림이 아니라 현재 상태 표시로 처리해야 한다. 그래서 `previous`가 connected였는지 같이 봐야 했다.

BLE도 비슷했다. 스캔 타임아웃은 화면에 안내 문구를 보여주는 값이고, 재연결 실패 후 권한 설정 화면을 여는 동작은 사건이다. 둘을 섞으면 테스트가 어려워진다. ProviderContainer 테스트에서는 상태 값만 검증하고, Widget test에서는 `listen`에 의해 SnackBar가 한 번만 뜨는지 확인하는 편이 깔끔했다.

주의할 점도 있다. `ref.listen` 안에서 무거운 로직을 직접 돌리면 안 된다. Crashlytics 기록, MQTT 재구독, BLE reconnect 같은 작업은 Repository나 Service로 넘기고 Widget에서는 호출만 해야 한다. 안 그러면 Riverpod으로 옮겼는데도 화면 파일이 다시 Controller처럼 커진다.

짧게 정리하면 이렇다. `watch`는 계속 보이는 값, `listen`은 한 번 처리할 사건이다. Flutter IoT 앱에서 MQTT 연결 끊김과 Flutter BLE 재연결 같은 문제를 이 기준으로 나누면 화면 코드가 꽤 조용해진다.
