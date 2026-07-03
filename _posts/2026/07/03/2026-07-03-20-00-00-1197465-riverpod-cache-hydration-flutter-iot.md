---
layout: post
title: "Flutter IoT Riverpod cache hydration - Flutter 스마트홈 첫 화면을 캐시로 즉시 채우기"
description: "Flutter IoT 앱에서 Riverpod과 Realm 오프라인 캐시로 Flutter 스마트홈 첫 화면을 즉시 그리고 MQTT와 Flutter BLE 실시간 상태로 보정한 기준을 정리했다."
date: 2026-07-03
tags: [Flutter, Riverpod, Realm, MQTT, BLE, IoT, 스마트홈, 오프라인캐시]
comments: true
share: true
---
![Flutter IoT Riverpod cache hydration](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)
이 그림에서는 스마트홈 대시보드가 서버 응답을 기다리지 않고 로컬 상태를 바로 보여줘야 하는 상황을 보면 된다.

Flutter IoT 앱에서 Riverpod으로 상태를 옮길 때 Flutter 스마트홈 첫 화면은 Realm 오프라인 캐시로 즉시 채우고, MQTT와 Flutter BLE 실시간 값으로 나중에 보정하는 쪽이 체감상 더 낫다. 네트워크가 1초만 늦어도 빈 화면부터 보이면 사용자는 앱이 멈춘 줄 안다. 처음엔 `FutureProvider` 하나에서 API, MQTT 연결, 로컬 DB 조회를 다 처리했는데 로딩 상태가 너무 자주 흔들렸다.

## 캐시는 임시 화면이 아니라 기준값이다

내가 헷갈렸던 건 캐시를 “오프라인일 때만 쓰는 예외”로 본 점이다. 스마트홈 앱에서는 마지막 보일러 온도, 전원 상태, Space 이름, 즐겨찾기 기기 순서가 전부 첫 화면 기준값이 된다. MQTT가 붙으면 최신값으로 바뀌고, BLE 등록 화면에 들어가면 주변 기기 스캔값이 따로 흐른다.

| 값 | 첫 화면 소스 | 갱신 소스 | 기준 |
|---|---|---|---|
| 기기 목록 | Realm | REST 또는 MQTT retained 상태 | 바로 표시 |
| 보일러 현재 온도 | Realm | MQTT telemetry | 오래된 표시 허용 |
| BLE 스캔 목록 | 없음 | flutter_blue_plus scan | 캐시 금지 |
| Space 권한 | Realm | REST | 실패 시 잠금 처리 |

여기서 BLE 스캔 목록을 캐시하지 않는 게 중요했다. 10분 전 주변에 있던 기기를 보여주면 등록 UX가 더 나빠진다. 반대로 보일러 온도는 2~3분 전 값이라도 “마지막 수신” 배지만 붙이면 빈 화면보다 낫다.

## Provider를 두 단계로 나눈다

아래 코드는 Realm 캐시를 즉시 읽고, MQTT 스트림이 들어오면 같은 상태를 덮어쓰는 구조다.

```dart
final cachedDashboardProvider =
    FutureProvider.autoDispose.family<DashboardState, String>((ref, spaceId) {
  final repository = ref.watch(deviceRepositoryProvider);
  return repository.loadCachedDashboard(spaceId);
});

final liveDashboardProvider =
    StreamProvider.autoDispose.family<DashboardState, String>((ref, spaceId) {
  final mqtt = ref.watch(mqttSessionProvider);
  return mqtt.watchDashboard(spaceId);
});

final dashboardViewProvider =
    Provider.autoDispose.family<AsyncValue<DashboardState>, String>((ref, spaceId) {
  final cached = ref.watch(cachedDashboardProvider(spaceId));
  final live = ref.watch(liveDashboardProvider(spaceId));

  return live.when(
    data: AsyncData.new,
    loading: () => cached,
    error: (error, stackTrace) => cached.hasValue
        ? AsyncData(cached.requireValue.copyWith(isStale: true))
        : AsyncError(error, stackTrace),
  );
});
```

핵심은 캐시와 실시간 값을 같은 Provider 안에서 억지로 섞지 않는 것이다. 캐시는 빠르게 읽는 값이고, MQTT는 연결 상태에 따라 늦거나 실패할 수 있는 값이다. 둘을 나누면 Widget에서는 `dashboardViewProvider(spaceId)`만 보면 된다.

## stale 표시를 빼먹으면 사고 난다

캐시 화면의 함정은 사용자가 최신 상태라고 착각하는 것이다. 실제로 MQTT 연결이 끊긴 상태에서 보일러 전원이 켜져 있다고 표시한 적이 있었다. 기기는 이미 꺼졌는데 앱에는 마지막 ON 상태가 남아 있었다. 그 뒤로 `isStale`, `lastUpdatedAt`, `connectionState`를 화면 모델에 같이 넣었다.

체크 기준은 짧게 잡았다.

- `lastUpdatedAt`이 5분을 넘으면 “마지막 수신” 문구를 보여준다.
- MQTT가 끊긴 상태에서 전원 버튼을 누르면 offline queue로 보내고 대기 상태를 표시한다.
- BLE 스캔, 페어링, 권한 상태는 캐시로 복원하지 않는다.
- Space 전환 시 이전 Space 캐시는 즉시 invalidate한다.

짧게 정리하면 이렇다. Riverpod cache hydration은 로딩을 숨기는 꼼수가 아니라, Flutter IoT 대시보드의 첫 기준값을 정하는 방식이다. Realm 캐시는 화면을 빨리 열기 위해 쓰고, MQTT와 Flutter BLE는 현재성을 보정하는 역할로 분리한다. 오래된 값에는 반드시 표시를 붙인다. 이 세 가지만 지켜도 첫 화면은 빨라지고, 상태가 틀렸을 때의 위험은 꽤 줄어든다.
