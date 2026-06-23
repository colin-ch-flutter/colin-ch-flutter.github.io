---
layout: post
title: "FOTA(무선 펌웨어 업데이트) - 앱에서 기기 펌웨어 업데이트"
description: " "
date: 2026-06-08
tags: [Flutter, FOTA, 펌웨어, OTA업데이트, IoT]
comments: true
share: true
---

# FOTA(무선 펌웨어 업데이트) - 앱에서 기기 펌웨어 업데이트

![펌웨어 업데이트](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## FOTA가 왜 필요한가

기기가 한 번 설치되면 나중에 버그를 고치거나 새 기능을 추가할 때 어떻게 할까? 예전에는 AS 기사가 직접 방문해서 노트북으로 업데이트했다. FOTA(Firmware Over The Air)는 인터넷을 통해 원격으로 업데이트한다.

사용자 입장에서는 앱에서 "업데이트 가능" 알림을 보고, 버튼 한 번으로 최신 펌웨어로 업그레이드할 수 있다.

## 업데이트 흐름

```
1. 앱이 서버에서 최신 펌웨어 버전 확인
2. 현재 기기 버전과 비교
3. 업데이트 가능하면 사용자에게 알림
4. 사용자가 업데이트 시작
5. 서버에서 기기로 MQTT 업데이트 명령 전송
6. 기기가 펌웨어 다운로드 및 적용
7. 기기 재시작
8. 앱이 완료 확인
```

## IoT DeviceScadaFotaView - 업데이트 목록

```dart
class IoT DeviceScadaFotaController extends GetxController {
  final String deviceId;
  
  final Rx<FotaStatus?> fotaStatus = Rx(null);
  final RxBool isLoading = false.obs;

  @override
  void onInit() {
    super.onInit();
    loadFotaStatus();
  }

  Future<void> loadFotaStatus() async {
    isLoading.value = true;
    final result = await _apiRepo.getFotaStatus(deviceId);
    isLoading.value = false;
    
    result.fold(
      onSuccess: (status) => fotaStatus.value = status,
      onFailure: (msg) => Get.snackbar('오류', msg),
    );
  }

  Future<void> startUpdate(String fotaId) async {
    // 업데이트 전 확인
    final confirmed = await Get.dialog<bool>(
      AlertDialog(
        title: const Text('펌웨어 업데이트'),
        content: const Text(
          '업데이트 중에는 기기를 끄지 마세요.\n'
          '업데이트는 약 5-10분 소요됩니다.',
        ),
        actions: [
          TextButton(onPressed: () => Get.back(result: false), child: const Text('취소')),
          ElevatedButton(onPressed: () => Get.back(result: true), child: const Text('시작')),
        ],
      ),
    );
    
    if (confirmed != true) return;
    
    Get.toNamed(Routes.iot_deviceScadaDetailFota, arguments: {
      'deviceId': deviceId,
      'fotaId': fotaId,
    });
  }
}
```

## IoT DeviceScadaDetailFotaView - 업데이트 진행 화면

```dart
class IoT DeviceScadaDetailFotaController extends GetxController {
  final String deviceId;
  final String fotaId;
  
  final RxDouble progress = 0.0.obs;
  final Rx<FotaUpdateState> state = FotaUpdateState.idle.obs;
  final RxString statusMessage = ''.obs;

  @override
  void onInit() {
    super.onInit();
    startUpdate();
    // MQTT로 진행 상황 수신
    _mqttRepo.fotaProgressStream
        .where((p) => p.deviceId == deviceId)
        .listen(_onProgressUpdate);
  }

  Future<void> startUpdate() async {
    state.value = FotaUpdateState.downloading;
    statusMessage.value = '펌웨어 다운로드 중...';
    
    await _apiRepo.triggerFotaUpdate(
      deviceId: deviceId,
      fotaId: fotaId,
    );
  }

  void _onProgressUpdate(FotaProgress update) {
    progress.value = update.percentage / 100.0;
    
    switch (update.stage) {
      case FotaStage.downloading:
        state.value = FotaUpdateState.downloading;
        statusMessage.value = '펌웨어 다운로드 중... (${update.percentage}%)';
        break;
      case FotaStage.installing:
        state.value = FotaUpdateState.installing;
        statusMessage.value = '펌웨어 설치 중... (${update.percentage}%)';
        break;
      case FotaStage.completed:
        state.value = FotaUpdateState.completed;
        statusMessage.value = '업데이트 완료! 기기가 재시작됩니다.';
        _handleCompleted();
        break;
      case FotaStage.failed:
        state.value = FotaUpdateState.failed;
        statusMessage.value = '업데이트 실패: ${update.errorMessage}';
        break;
    }
  }

  void _handleCompleted() {
    Future.delayed(const Duration(seconds: 3), () {
      Get.back();
      Get.snackbar('완료', '펌웨어 업데이트가 완료됐습니다');
    });
  }
}
```

## 진행 UI

```dart
// View
Obx(() {
  return Column(
    children: [
      // 애니메이션 아이콘
      if (controller.state.value == FotaUpdateState.completed)
        const Icon(Icons.check_circle, color: Colors.green, size: 80)
      else if (controller.state.value == FotaUpdateState.failed)
        const Icon(Icons.error, color: Colors.red, size: 80)
      else
        const CircularProgressIndicator(),
      
      const SizedBox(height: 24),
      Text(controller.statusMessage.value),
      const SizedBox(height: 16),
      
      // 진행 바
      if (controller.state.value != FotaUpdateState.completed &&
          controller.state.value != FotaUpdateState.failed)
        LinearProgressIndicator(value: controller.progress.value),
    ],
  );
})
```

## 실패 처리

업데이트 실패 시 기기가 이전 버전으로 롤백되는지 확인이 중요하다. 좋은 펌웨어는 업데이트 실패 시 자동으로 이전 버전으로 복구된다. 앱에서는 실패 후 기기 상태를 다시 조회해서 정상 동작하는지 확인한다.

---

다음 편은 EMS(에너지 관리 시스템) 화면 구현이다.
