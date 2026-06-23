---
layout: post
title: "GetX Controller로 기기 제어 화면 구현하기"
description: " "
date: 2026-05-23
tags: [Flutter, GetX, UI, IoT 기기제어, IoT]
comments: true
share: true
---

# GetX Controller로 기기 제어 화면 구현하기

![스마트홈 컨트롤](https://images.unsplash.com/photo-1585771724684-38269d6639fd?w=800&q=80)

## 제어 화면의 요구사항

기기 제어 화면은 IoT 앱의 핵심이다. 요구사항을 정리하면:

- 현재 온도 / 설정 온도 표시
- 전원 On/Off 버튼
- 운전 모드 전환 (난방, 온수, 외출)
- 온도 +/- 조절 버튼
- 기기 오프라인/오류 상태 표시
- MQTT 명령 발행 후 응답까지 로딩 표시

## Controller 구현

```dart
class IoT DeviceControlController extends GetxController {
  final String deviceId;
  final IoT DeviceMqttRepository _mqttRepo;
  final AppCache _cache;

  IoT DeviceControlController({
    required this.deviceId,
    required IoT DeviceMqttRepository mqttRepo,
    required AppCache cache,
  })  : _mqttRepo = mqttRepo,
        _cache = cache;

  // 상태 변수
  final Rx<IoT DeviceStatus?> status = Rx(null);
  final RxBool isCommandPending = false.obs;

  @override
  void onInit() {
    super.onInit();
    // 캐시된 상태로 초기값 설정
    status.value = _cache.loadIoT DeviceStatus(deviceId);
    // MQTT 스트림 구독
    _mqttRepo.statusStream
        .where((s) => s.deviceId == deviceId)
        .listen(_onStatusUpdated);
  }

  void _onStatusUpdated(IoT DeviceStatus newStatus) {
    status.value = newStatus;
    isCommandPending.value = false;
    _cache.saveIoT DeviceStatus(deviceId, newStatus);
  }

  Future<void> togglePower() async {
    final current = status.value;
    if (current == null) return;

    isCommandPending.value = true;
    await _mqttRepo.publishPowerControl(deviceId, !current.isOn);
    
    // 10초 후에도 응답 없으면 로딩 해제
    Future.delayed(const Duration(seconds: 10), () {
      if (isCommandPending.value) isCommandPending.value = false;
    });
  }

  Future<void> increaseTemperature() async {
    final current = status.value;
    if (current == null) return;
    if (current.targetTemp >= 80) return; // 최대값

    isCommandPending.value = true;
    await _mqttRepo.publishTemperatureControl(
      deviceId,
      (current.targetTemp + 1).toInt(),
    );
  }

  Future<void> decreaseTemperature() async {
    final current = status.value;
    if (current == null) return;
    if (current.targetTemp <= 10) return; // 최소값

    isCommandPending.value = true;
    await _mqttRepo.publishTemperatureControl(
      deviceId,
      (current.targetTemp - 1).toInt(),
    );
  }

  Future<void> changeMode(IoT DeviceMode mode) async {
    isCommandPending.value = true;
    await _mqttRepo.publishModeControl(deviceId, mode);
  }
}
```

## View 구현

```dart
class IoT DeviceControlView extends GetView<IoT DeviceControlController> {
  const IoT DeviceControlView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('IoT 기기 제어')),
      body: Obx(() {
        final status = controller.status.value;
        final isPending = controller.isCommandPending.value;

        if (status == null) {
          return const Center(child: CircularProgressIndicator());
        }

        return Stack(
          children: [
            Column(
              children: [
                // 온도 표시
                _TemperatureDisplay(status: status),
                const SizedBox(height: 32),
                // 온도 조절 버튼
                _TemperatureControls(
                  onIncrease: controller.increaseTemperature,
                  onDecrease: controller.decreaseTemperature,
                  enabled: status.isOn && !isPending,
                ),
                const SizedBox(height: 24),
                // 운전 모드 선택
                _ModeSelector(
                  currentMode: status.mode,
                  onModeChanged: controller.changeMode,
                ),
                const Spacer(),
                // 전원 버튼
                _PowerButton(
                  isOn: status.isOn,
                  onPressed: controller.togglePower,
                ),
              ],
            ),
            // 커맨드 대기 중 오버레이
            if (isPending)
              const _LoadingOverlay(),
          ],
        );
      }),
    );
  }
}
```

## 온도 표시 위젯

```dart
class _TemperatureDisplay extends StatelessWidget {
  final IoT DeviceStatus status;

  const _TemperatureDisplay({required this.status});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(
          '${status.currentTemp.toStringAsFixed(1)}°C',
          style: Theme.of(context).textTheme.displayLarge,
        ),
        Text(
          '목표: ${status.targetTemp.toInt()}°C',
          style: Theme.of(context).textTheme.bodyMedium?.copyWith(
            color: Theme.of(context).colorScheme.secondary,
          ),
        ),
        if (status.errorCode != null)
          Padding(
            padding: const EdgeInsets.only(top: 8),
            child: Chip(
              label: Text('오류: ${status.errorCode}'),
              backgroundColor: Colors.red.shade100,
            ),
          ),
      ],
    );
  }
}
```

## 버튼 연속 입력 방지

온도 +/- 버튼을 빠르게 누르면 명령이 중복으로 발행된다. 디바운싱으로 처리한다:

```dart
Timer? _debounceTimer;
int _pendingTempChange = 0;

void _onTempButtonPressed(int delta) {
  _pendingTempChange += delta;
  _debounceTimer?.cancel();
  _debounceTimer = Timer(const Duration(milliseconds: 300), () {
    if (_pendingTempChange != 0) {
      final newTemp = (status.value!.targetTemp + _pendingTempChange).toInt();
      _pendingTempChange = 0;
      _mqttRepo.publishTemperatureControl(deviceId, newTemp.clamp(10, 80));
    }
  });
}
```

---

다음 편은 fl_chart로 에너지 사용량을 시각화하는 방법이다.
