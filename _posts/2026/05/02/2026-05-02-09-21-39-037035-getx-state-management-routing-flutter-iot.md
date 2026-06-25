---
layout: post
title: "GetX 완전 정복 - Flutter IoT 앱의 상태관리와 라우팅"
description: " "
date: 2026-05-02
tags: [Flutter, GetX, 상태관리, 라우팅]
comments: true
share: true
---
![코드 에디터](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## 왜 GetX를 선택했나

Flutter 상태관리 라이브러리는 선택지가 너무 많다. Provider, Riverpod, Bloc, MobX, GetX... 처음 기술 선택을 할 때 한참 고민했다.

최종적으로 GetX를 선택한 이유는 크게 두 가지였다.

**첫째, 라우팅과 상태관리를 하나로 처리한다.** IoT 앱은 화면 전환이 꽤 많다. 기기 등록 플로우만 해도 안내 → BLE 스캔 → 연결 → WiFi 입력 → 완료 순으로 여러 화면을 거친다. GetX는 라우팅 자체가 내장되어 있어서 별도 Navigator 설정 없이 `Get.toNamed('/device_connect_guide1')` 한 줄로 이동한다.

**둘째, iot_deviceplate가 적다.** Bloc은 Event, State, Bloc 클래스를 모두 만들어야 하는데, GetX는 Controller 하나로 시작할 수 있다.

## Controller, Binding, GetPage 구조

GetX의 핵심 세 가지다.

### Controller

비즈니스 로직과 상태를 담는 곳이다:

```dart
class IoT DeviceControlController extends GetxController {
  final IoT DeviceMqttRepository _mqttRepo;

  // Rx 변수로 상태를 observable하게 만든다
  final Rx<IoT DeviceStatus?> status = Rx(null);
  final RxBool isLoading = false.obs;

  IoT DeviceControlController(this._mqttRepo);

  @override
  void onInit() {
    super.onInit();
    // MQTT 스트림을 구독
    _mqttRepo.statusStream.listen(_onStatusReceived);
  }

  void _onStatusReceived(IoT DeviceStatus newStatus) {
    status.value = newStatus;
  }

  Future<void> togglePower() async {
    isLoading.value = true;
    try {
      final isOn = !(status.value?.isOn ?? false);
      await _mqttRepo.publishPowerControl(deviceId, isOn);
    } finally {
      isLoading.value = false;
    }
  }
}
```

### Binding

화면에 필요한 의존성을 주입하는 역할이다. 화면이 열릴 때 자동으로 실행된다:

```dart
class IoT DeviceControlBinding extends Bindings {
  @override
  void dependencies() {
    // Repository는 이미 등록되어 있다고 가정
    final mqttRepo = Get.find<IoT DeviceMqttRepository>();
    Get.lazyPut(() => IoT DeviceControlController(mqttRepo));
  }
}
```

### GetPage

라우팅 설정이다. `app_pages.dart` 에 모든 라우트를 등록해둔다:

```dart
GetPage(
  name: Routes.iot_deviceControl,
  page: () => const IoT DeviceControlView(),
  binding: IoT DeviceControlBinding(),
),
```

이렇게 하면 View, Controller, 의존성 주입이 깔끔하게 분리된다.

## Rx 변수 활용법

GetX의 반응형 상태관리는 Rx 변수를 쓴다. 기본 타입 확장으로 `.obs`를 붙이면 된다:

```dart
RxBool isOn = false.obs;        // bool
RxInt temperature = 22.obs;     // int
RxString nickname = ''.obs;     // String
Rx<IoT DeviceMode> mode = IoT DeviceMode.heating.obs;  // enum
RxList<Device> devices = <Device>[].obs;        // List
```

View에서는 `Obx`로 감싸면 값이 바뀔 때 자동으로 리빌드된다:

```dart
Obx(() {
  final status = controller.status.value;
  if (status == null) return const LoadingWidget();
  
  return Column(
    children: [
      Text('현재 온도: ${status.currentTemp}°C'),
      Switch(
        value: status.isOn,
        onChanged: (_) => controller.togglePower(),
      ),
    ],
  );
})
```

## IoT 앱에서 특히 유용한 패턴

MQTT 메시지처럼 외부에서 계속 상태가 들어오는 앱에서 GetX가 특히 빛을 발한다. MQTT 스트림을 Controller에서 구독하고, 값이 들어오면 Rx 변수를 업데이트하기만 하면 UI가 알아서 반응한다.

```dart
// Controller
ever(status, (IoT DeviceStatus? s) {
  // 상태가 바뀔 때마다 부가 작업 처리
  if (s?.errorCode != null) {
    Get.snackbar('경고', '기기 오류: ${s!.errorCode}');
  }
});
```

`ever`는 변수가 바뀔 때마다 콜백을 실행하는 GetX 유틸이다. 에러 코드가 들어올 때 스낵바를 띄우는 데 유용하다.

## 주의할 점

GetX를 쓰다 보면 Controller에 너무 많은 걸 몰아넣게 되는 함정이 있다. API 호출, 비즈니스 로직, UI 상태가 한 파일에 다 들어가면 Controller가 비대해진다.

깔끔하게 유지하려면:
- Controller는 UI 상태만 관리
- 비즈니스 로직은 Repository나 UseCase로 분리
- API 호출은 Repository에서

이 원칙을 지키면 나중에 코드 찾기가 훨씬 편해진다.

---

다음 편은 Easy Localization으로 3개 국어를 지원한 과정이다.
