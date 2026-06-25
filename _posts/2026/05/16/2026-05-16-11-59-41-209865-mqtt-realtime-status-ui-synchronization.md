---
layout: post
title: "MQTT 실시간 상태 수신과 UI 동기화"
description: " "
date: 2026-05-16
tags: [Flutter, MQTT, GetX, 실시간, 상태관리]
comments: true
share: true
---
![실시간 대시보드](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

## 데이터 흐름 전체 그림

```
[IoT 기기 기기]
    ↓ MQTT Publish
[AWS IoT Core]
    ↓ MQTT Deliver
[IotMqttService] (raw JSON)
    ↓ 파싱
[MqttMapper] (data model → domain model)
    ↓ 변환
[IoT DeviceMqttRepositoryImpl] (domain model)
    ↓ Stream
[IoT DeviceControlController] (Rx 변수 업데이트)
    ↓ Obx
[IoT DeviceControlView] (화면 자동 갱신)
```

이 파이프라인이 한 번에 설계되면 MQTT 메시지 하나가 IoT 기기에서 화면까지 자동으로 흐른다.

## Mapper 구현

MQTT로 받은 JSON을 도메인 모델로 변환한다:

```dart
// data/mapper/mqtt/iot_device_status_mapper.dart
class IoT DeviceStatusMqttMapper {
  static IoT DeviceStatus fromJson(Map<String, dynamic> json) {
    return IoT DeviceStatus(
      deviceId: json['deviceId'] as String,
      isOn: json['power'] == 'on',
      currentTemp: (json['currentTemp'] as num).toDouble(),
      targetTemp: (json['targetTemp'] as num).toDouble(),
      mode: IoT DeviceMode.fromString(json['mode'] as String),
      errorCode: json['errorCode'] as String?,
      updatedAt: DateTime.fromMillisecondsSinceEpoch(json['timestamp'] as int),
    );
  }
}
```

## Repository에서 Stream 제공

```dart
// data/repositories/iot_device_mqtt_repository_impl.dart
class IoT DeviceMqttRepositoryImpl implements IoT DeviceMqttRepository {
  final IotMqttService _mqttService;
  final _statusController = StreamController<IoT DeviceStatus>.broadcast();

  IoT DeviceMqttRepositoryImpl(this._mqttService) {
    _mqttService.messageStream.listen(_handleMqttMessage);
  }

  @override
  Stream<IoT DeviceStatus> get statusStream => _statusController.stream;

  void _handleMqttMessage(MqttReceivedMessage message) {
    final topic = message.topic;
    final payload = _extractPayload(message);

    if (topic.endsWith('/status')) {
      try {
        final json = jsonDecode(payload) as Map<String, dynamic>;
        final status = IoT DeviceStatusMqttMapper.fromJson(json);
        _statusController.add(status);
      } catch (e) {
        // 파싱 실패는 조용히 넘긴다
        KdLog.e('상태 파싱 오류: $e');
      }
    }
  }
}
```

## Controller에서 Stream 구독

```dart
class IoT DeviceControlController extends GetxController {
  final IoT DeviceMqttRepository _mqttRepo;
  StreamSubscription<IoT DeviceStatus>? _statusSub;

  final Rx<IoT DeviceStatus?> status = Rx(null);
  final RxBool isLoading = false.obs;

  IoT DeviceControlController(this._mqttRepo);

  @override
  void onInit() {
    super.onInit();
    _statusSub = _mqttRepo.statusStream
        .where((s) => s.deviceId == deviceId)
        .listen(_onStatusUpdated);
  }

  void _onStatusUpdated(IoT DeviceStatus newStatus) {
    status.value = newStatus;
    isLoading.value = false; // 응답 왔으면 로딩 끝
  }

  @override
  void onClose() {
    _statusSub?.cancel(); // 메모리 누수 방지
    super.onClose();
  }
}
```

`onClose()`에서 구독을 취소하는 게 중요하다. 화면이 닫혔는데 구독이 남아있으면 메모리 누수가 생긴다.

## View에서 Obx로 자동 갱신

```dart
class IoT DeviceControlView extends GetView<IoT DeviceControlController> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Obx(() {
        final status = controller.status.value;
        
        if (status == null) {
          return const Center(child: CircularProgressIndicator());
        }
        
        return Column(
          children: [
            // 현재 온도
            Text(
              '${status.currentTemp.toStringAsFixed(1)}°C',
              style: const TextStyle(fontSize: 48, fontWeight: FontWeight.bold),
            ),
            // 전원 버튼
            Switch(
              value: status.isOn,
              onChanged: (value) {
                controller.isLoading.value = true;
                controller.togglePower();
              },
            ),
            // 에러 표시
            if (status.errorCode != null)
              ErrorBanner(code: status.errorCode!),
          ],
        );
      }),
    );
  }
}
```

## 여러 기기 동시 상태 관리

여러 기기를 메인 화면에서 동시에 보여줄 때:

```dart
// 기기별 상태를 Map으로 관리
final RxMap<String, IoT DeviceStatus> deviceStatuses = <String, IoT DeviceStatus>{}.obs;

void _onStatusUpdated(IoT DeviceStatus status) {
  deviceStatuses[status.deviceId] = status;
}
```

```dart
// 기기 목록 UI
Obx(() {
  return ListView.builder(
    itemCount: controller.devices.length,
    itemBuilder: (context, index) {
      final device = controller.devices[index];
      final status = controller.deviceStatuses[device.id];
      
      return DeviceCard(
        device: device,
        status: status,  // null이면 오프라인 표시
      );
    },
  );
})
```

## 상태 캐싱

앱을 재시작했을 때 MQTT 메시지가 오기 전까지 빈 화면이 보이면 좋지 않다. 마지막 상태를 캐시에 저장해두고 초기값으로 사용한다:

```dart
void _onStatusUpdated(IoT DeviceStatus status) {
  status.value = status;
  _cache.saveIoT DeviceStatus(status.deviceId, status); // 캐시 저장
}

// 앱 시작 시 캐시에서 복원
void onInit() {
  final cached = _cache.loadIoT DeviceStatus(deviceId);
  if (cached != null) {
    status.value = cached;
  }
  // 이후 MQTT 메시지가 오면 실제 상태로 갱신
}
```

---

다음 편은 MQTT 연결을 안정적으로 유지하는 방법이다.
