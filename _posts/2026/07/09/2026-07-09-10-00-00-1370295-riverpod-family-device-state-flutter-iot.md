---
layout: post
title: "Flutter IoT Riverpod family - BLE 기기별 상태가 섞이지 않게 나누기"
description: "Flutter IoT 앱에서 Riverpod family로 Flutter BLE 기기별 연결 상태와 스마트홈 제어 상태를 분리해 상태 오염을 줄인 기준이다."
date: 2026-07-09
tags: [Flutter, Riverpod, BLE, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod family 기기별 상태 분리](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
이 그림에서는 여러 스마트홈 기기 상태가 한 화면에 모여 있어도 내부 상태는 기기별로 분리되어야 한다는 점을 보면 된다.

Flutter IoT 앱에서 Flutter BLE 상태를 Riverpod `family`로 나누지 않으면 스마트홈 기기 A의 연결 실패가 기기 B 화면에 묻어 나오는 일이 생겼다. 처음엔 `selectedDeviceProvider` 하나로 충분하다고 봤다. 화면에서 선택한 기기만 바뀌니까 상태도 따라 바뀔 거라 생각했다. 해보니 아니었다. BLE 연결, MQTT reported 상태, 사용자가 누른 pending 명령은 모두 수명이 조금씩 달랐다.

[BLE 재연결 전략]({% post_url 2026-05-11-09-24-36-148140-ble-connection-stability-reconnect-error-handling %})을 만들 때도 비슷했다. 연결 실패 자체보다 “어느 기기의 실패인가”를 놓치면 로그가 쓸모없어진다. 특히 보일러 2대, 온도센서 3대가 등록된 집에서는 같은 `disconnected`라도 의미가 다르다. 하나는 전원이 빠진 상태고, 다른 하나는 앱이 백그라운드에서 돌아오며 잠깐 끊긴 상태일 수 있다.

## selectedDevice 하나로는 부족했다

내가 겪은 문제는 화면 전환보다 빠른 이벤트였다. 사용자가 거실 보일러 상세 화면에서 전원을 누르고 바로 안방 보일러로 이동하면 MQTT 응답은 늦게 도착한다. 이때 전역 command 상태를 쓰면 안방 화면 버튼이 로딩으로 보였다. 숫자로 보면 LTE 환경에서 reported 응답이 1.4초 늦게 온 적이 있었고, 화면 이동은 300ms 안에 끝났다.

| 상태 | 전역 Provider | `family(deviceId)` |
|---|---|---|
| BLE 연결 | 마지막 기기 상태로 덮임 | 기기별 연결 상태 유지 |
| MQTT reported | 늦은 응답이 현재 화면에 반영됨 | 응답 대상 기기에만 반영 |
| 버튼 pending | 다른 기기 버튼까지 잠김 | 누른 기기만 잠김 |
| 로그 추적 | 원인 구분이 흐림 | deviceId 기준으로 추적 가능 |

## 기기 ID를 상태의 경계로 둔다

Repository는 여전히 공통으로 둔다. 대신 UI가 읽는 상태 Provider는 `deviceId`를 인자로 받게 했다. 이 기준을 세우니 GetX Controller에서 `currentDevice`, `previousDevice`, `pendingDeviceId` 같은 필드가 늘어나던 문제가 줄었다.

기기별 reported 상태를 읽는 Provider는 이런 형태로 뒀다.

```dart
final deviceStatusProvider =
    StreamProvider.autoDispose.family<DeviceStatus, String>((ref, deviceId) {
  final repository = ref.watch(deviceRepositoryProvider);
  return repository.watchStatus(deviceId);
});
```

쓰기 명령도 같은 경계를 따른다. 낙관적 UI를 무리하게 켜기보다, 누른 기기의 pending만 따로 잡는 쪽이 덜 위험했다.

```dart
final deviceCommandProvider =
    NotifierProvider.family<DeviceCommandNotifier, CommandState, String>(
  DeviceCommandNotifier.new,
);

class DeviceCommandNotifier extends FamilyNotifier<CommandState, String> {
  @override
  CommandState build(String deviceId) => const CommandState.idle();

  Future<void> togglePower(bool value) async {
    state = const CommandState.pending();
    final repository = ref.read(deviceRepositoryProvider);

    try {
      await repository.setPower(arg, value);
      state = const CommandState.idle();
    } catch (error) {
      state = CommandState.failed(error.toString());
    }
  }
}
```

여기서 헷갈렸던 건 `arg`다. `FamilyNotifier` 안에서는 생성자 인자를 따로 저장하지 않아도 `arg`로 접근할 수 있다. 처음엔 필드에 `deviceId`를 복사해뒀는데, 테스트에서 provider override를 바꿀 때 오히려 헷갈렸다.

## autoDispose는 화면 기준으로만 보지 않는다

`autoDispose`를 붙이면 화면을 나갈 때 정리돼서 편하다. 다만 BLE 연결처럼 화면보다 오래 살아야 하는 상태까지 무조건 정리하면 재진입 때 reconnect가 반복된다. 내가 확인한 기준으론 표시 상태와 명령 상태는 `autoDispose`가 맞았고, 실제 BLE 세션은 별도 서비스나 상위 Provider에 두는 편이 안정적이었다.

체크 기준은 단순하게 잡았다.

- 화면에만 필요한 값이면 `autoDispose.family`
- 실제 통신 세션이면 앱 수명 Provider
- 기기별 버튼 로딩이면 command `family`
- 여러 기기를 합쳐 보여주면 dashboard Provider에서 조합

[클린 아키텍처 구조]({% post_url 2026-05-01-11-14-26-024690-clean-architecture-large-flutter-project-structure %})로 보면 `family`는 Repository를 쪼개는 도구가 아니다. 같은 Repository를 읽되, UI 상태의 주소를 기기 ID로 나누는 도구에 가깝다. 이 차이를 놓치면 Provider가 많아진 것처럼 보이지만, 실제로는 전역 상태 하나에 if문을 쌓는 것보다 디버깅이 쉬웠다.

짧게 정리하면 이렇다.

- Flutter IoT 화면에서 기기별 상태는 `family(deviceId)`로 분리하는 편이 안전하다.
- Flutter BLE 재연결과 MQTT reported 응답은 현재 화면보다 늦게 도착할 수 있다.
- `autoDispose`는 표시 상태에 쓰고, 실제 통신 세션 수명은 별도로 판단한다.
- Riverpod `family`를 쓰면 스마트홈 다중 기기 화면에서 상태 오염을 줄일 수 있다.
