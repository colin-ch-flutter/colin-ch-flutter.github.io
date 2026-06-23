---
layout: post
title: "Flutter 앱 성능 최적화 - 불필요한 리빌드 줄이기"
description: " "
date: 2026-06-13
tags: [Flutter, 성능최적화, GetX, 리빌드, DevTools]
comments: true
share: true
---

# Flutter 앱 성능 최적화 - 불필요한 리빌드 줄이기

![성능 최적화](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## IoT 앱에서 성능이 중요한 이유

MQTT로 초당 여러 번 상태 업데이트가 오는 앱에서 성능 관리를 소홀히 하면 문제가 생긴다. 각 업데이트마다 전체 화면이 리빌드되면 프레임 드롭이 발생하고 배터리도 많이 소모된다.

## Obx vs GetBuilder

GetX에서 상태를 구독하는 방법이 두 가지다:

**Obx**: 클로저 안에서 참조한 Rx 변수가 바뀌면 리빌드
```dart
Obx(() => Text(controller.temperature.value.toString()))
```

**GetBuilder**: `update()`를 명시적으로 호출할 때만 리빌드
```dart
GetBuilder<MyController>(
  builder: (c) => Text(c.temperature.toString()),
)
```

실시간으로 계속 바뀌는 값은 `Obx`, 사용자 액션에만 반응하는 값은 `GetBuilder`가 효율적이다.

## 세분화된 Rx 구독

전체 상태 객체를 하나의 Rx로 관리하면 상태 일부가 바뀔 때도 전체가 리빌드된다:

```dart
// 나쁜 예: 온도만 바뀌어도 전체가 리빌드됨
final Rx<IoT DeviceStatus> status = IoT DeviceStatus.empty().obs;

Obx(() => Column(
  children: [
    Text(controller.status.value.temperature.toString()), // 온도
    Text(controller.status.value.mode.name),               // 모드
    Text(controller.status.value.errorCode ?? ''),         // 오류
  ],
))
```

```dart
// 좋은 예: 각각 독립적으로 구독
final RxDouble temperature = 0.0.obs;
final Rx<IoT DeviceMode> mode = IoT DeviceMode.heating.obs;
final RxString? errorCode = Rxn<String>();

// 각 위젯이 필요한 값만 구독
Obx(() => Text(controller.temperature.value.toString()))
Obx(() => Text(controller.mode.value.name))
```

## const 위젯 활용

변하지 않는 위젯은 `const`로 만들면 리빌드에서 제외된다:

```dart
// 나쁜 예
Column(
  children: [
    SizedBox(height: 16),     // 재생성됨
    Divider(color: Colors.grey), // 재생성됨
    Icon(Icons.thermostat),   // 재생성됨
    Obx(() => Text(controller.temperature.value.toString())), // 변경 시 갱신
  ],
)

// 좋은 예
const Column(
  children: [
    SizedBox(height: 16),         // const → 재사용
    Divider(color: Colors.grey),  // const → 재사용
    Icon(Icons.thermostat),       // const → 재사용
  ],
)
// 변하는 부분만 Obx로 따로
Obx(() => Text(controller.temperature.value.toString()))
```

## ListView 최적화

긴 리스트는 `ListView.builder`를 쓴다:

```dart
// 나쁜 예: 모든 아이템을 한 번에 생성
ListView(
  children: devices.map((d) => DeviceCard(device: d)).toList(),
)

// 좋은 예: 보이는 아이템만 생성
ListView.builder(
  itemCount: devices.length,
  itemBuilder: (context, index) => DeviceCard(device: devices[index]),
)
```

아이템 크기가 고정이면 `itemExtent`를 지정하면 더 빠르다:

```dart
ListView.builder(
  itemCount: devices.length,
  itemExtent: 72, // 고정 높이
  itemBuilder: (context, index) => DeviceCard(device: devices[index]),
)
```

## Flutter DevTools로 프로파일링

어떤 위젯이 얼마나 자주 리빌드되는지 확인하는 방법:

```dart
// main.dart에서 디버그 모드 설정
if (kDebugMode) {
  debugPrintRebuildDirtyWidgets = true; // 리빌드되는 위젯 로그
}
```

DevTools의 Performance 탭에서 실제 프레임 렌더링 시간을 확인할 수 있다. 16ms(60fps)를 넘는 프레임이 있으면 최적화가 필요하다.

## 이미지 캐싱

네트워크 이미지를 매번 다시 다운로드하면 느리다:

```dart
// cached_network_image 패키지 (필요 시 추가)
CachedNetworkImage(
  imageUrl: profileImageUrl,
  placeholder: (context, url) => const CircularProgressIndicator(),
  errorWidget: (context, url, error) => const Icon(Icons.person),
)
```

---

다음 편은 WakelockPlus로 화면 켜진 채 유지하는 방법이다.
