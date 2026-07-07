---
layout: post
title: "Flutter IoT Riverpod selectAsync - 스마트홈 대시보드 부분 로딩 분리하기"
description: "Flutter IoT 앱에서 Riverpod selectAsync로 Flutter 스마트홈 대시보드의 MQTT 상태와 Flutter BLE 연결 정보를 부분 로딩으로 나누는 기준을 정리했다."
date: 2026-07-08
tags: [Flutter, Riverpod, MQTT, BLE, IoT, 스마트홈, CleanArchitecture]
comments: true
share: true
---

![Flutter IoT Riverpod selectAsync 대시보드 부분 로딩](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)
이 그림에서는 스마트홈 대시보드가 한 번에 뜨는 화면이 아니라 여러 비동기 상태가 조립된 결과라는 점을 보면 된다.

Flutter IoT 앱에서 Riverpod `selectAsync`는 Flutter 스마트홈 대시보드 전체를 `loading`으로 잠그지 않게 해준다. MQTT 상태 하나가 늦거나 Flutter BLE 근접 정보가 아직 안 왔다는 이유로 보일러 카드 전체를 스켈레톤으로 돌리는 건 실제 사용감이 별로였다. 내가 확인한 기준으론 대시보드 원본은 `AsyncValue`로 유지하고, 화면에 필요한 조각만 `selectAsync`로 뽑는 쪽이 덜 흔들렸다.

[이전 `select` 글]({% post_url 2026-07-01-18-00-00-1061670-riverpod-select-iot-dashboard-rebuild %})에서는 리빌드 범위를 줄이는 데 집중했다. [AsyncValue 글]({% post_url 2026-07-02-14-00-00-1111050-riverpod-asyncvalue-iot-dashboard-state %})에서는 로딩과 에러를 한 타입으로 묶었다. 그런데 둘을 같이 쓰는 지점에서 한 번 헷갈렸다. `ref.watch(dashboardProvider)`를 화면 상단, 카드, BLE 배너가 전부 읽으면 API 응답 하나가 늦을 때 모든 영역이 같이 멈춘다.

내가 나눈 기준은 이렇다.

| 화면 영역 | 필요한 값 | 처리 방식 |
| --- | --- | --- |
| 상단 Space 이름 | `space.name` | 캐시 우선 표시 |
| 보일러 카드 | `device.temperature`, `power` | `selectAsync`로 카드 값만 구독 |
| MQTT 배지 | `mqtt.connected` | 세션 Provider 별도 구독 |
| BLE 등록 배너 | `nearbyBleCount` | 실패해도 대시보드는 유지 |

처음 시도한 코드는 대시보드 상태에서 카드에 필요한 값만 기다리게 만드는 형태였다.

```dart
final dashboardProvider =
    FutureProvider.family<DashboardState, String>((ref, spaceId) async {
  final repository = ref.watch(dashboardRepositoryProvider);
  return repository.load(spaceId);
});

final boilerCardProvider =
    FutureProvider.family<BoilerCardState, String>((ref, deviceId) {
  return ref.watch(
    dashboardProvider(deviceId).selectAsync((dashboard) {
      final device = dashboard.devices.firstWhere((e) => e.id == deviceId);
      return BoilerCardState(
        name: device.name,
        power: device.power,
        temperature: device.temperature,
      );
    }),
  );
});
```

문제는 `dashboardProvider(spaceId)`를 써야 하는데 예제처럼 `deviceId`를 넘긴 부분이었다. 실제 코드에서는 `spaceId`와 `deviceId`를 같이 받는 파라미터 객체로 바꿨다. IoT 앱은 기기 ID만으로 화면을 만들 수 있는 것처럼 보여도, MQTT topic과 캐시 키는 대부분 Space 기준으로 묶인다.

수정한 형태는 조금 더 길지만 의도가 분명하다.

```dart
class DeviceQuery {
  final String spaceId;
  final String deviceId;

  const DeviceQuery(this.spaceId, this.deviceId);
}

final boilerCardProvider =
    FutureProvider.family<BoilerCardState, DeviceQuery>((ref, query) {
  return ref.watch(
    dashboardProvider(query.spaceId).selectAsync((dashboard) {
      final device =
          dashboard.devices.firstWhere((e) => e.id == query.deviceId);
      return BoilerCardState.fromDevice(device);
    }),
  );
});
```

여기서 `selectAsync`가 해결해준 건 로딩 UI의 범위다. Space 이름은 캐시로 바로 보여주고, 보일러 카드만 잠깐 로딩할 수 있다. BLE 등록 배너가 실패해도 MQTT 카드가 사라지지 않는다. 사용자는 "앱이 멈췄다"고 느끼지 않고, 어떤 영역이 아직 준비 중인지 볼 수 있다.

주의할 점도 있다. `selectAsync`를 너무 잘게 쪼개면 Provider 수가 늘고 테스트도 번거로워진다. 온도, 전원, 모드까지 각각 Provider로 만들면 MQTT 패킷 하나를 따라가기가 더 어려웠다. 나는 카드 단위까지만 나누고, 카드 내부 필드는 일반 모델로 묶었다.

짧게 정리하면 이렇다.

- Flutter 스마트홈 대시보드 전체를 한 `AsyncValue` 로딩에 묶으면 화면이 쉽게 멈춘 것처럼 보인다.
- Riverpod `selectAsync`는 비동기 원본에서 화면 조각만 기다리게 만들 때 쓴다.
- MQTT 세션, Flutter BLE 배너, 기기 카드는 수명과 실패 조건이 다르므로 Provider 경계를 나누는 편이 낫다.
- 너무 작은 필드 단위 Provider는 디버깅 비용을 키운다. IoT 화면에서는 카드 단위가 적당했다.
