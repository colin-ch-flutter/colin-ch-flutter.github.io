---
layout: post
title: "퀵 모드 기능 설계 - 자주 쓰는 설정 원터치로"
description: " "
date: 2026-05-28
tags: [Flutter, 퀵모드, UX, 스마트홈, 기능설계]
comments: true
share: true
---
![스마트홈 퀵설정](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

## 퀵 모드란

퀵 모드는 자주 사용하는 설정 조합을 프리셋으로 저장해두고, 원터치로 여러 기기에 동시 적용하는 기능이다.

예를 들어 "외출 모드"를 만들어두면, 버튼 하나로 모든 방의 IoT 기기를 외출 모드로 전환할 수 있다. "취침 모드"를 만들면 온도를 낮추고 조용한 운전으로 전환한다.

특히 여러 기기가 있을 때 빛을 발한다. 방마다 기기가 있다면 개별로 설정을 바꾸는 건 번거롭다.

## 데이터 구조

```dart
// domain/models/quick_mode.dart
class QuickMode {
  final String id;
  final String name;
  final String? iconName;
  final List<QuickModeDevice> devices;
  final int sortOrder;

  const QuickMode({
    required this.id,
    required this.name,
    this.iconName,
    required this.devices,
    required this.sortOrder,
  });
}

class QuickModeDevice {
  final String deviceId;
  final String deviceNickname;
  final IoT DeviceMode mode;
  final int? targetTemp;

  const QuickModeDevice({
    required this.deviceId,
    required this.deviceNickname,
    required this.mode,
    this.targetTemp,
  });
}
```

## 퀵 모드 추가 플로우

퀵 모드 추가 화면은 여러 단계로 구성된다:

1. **이름 입력 + 아이콘 선택** (`QuickAddView`)
2. **기기 선택** (`QuickDeviceAddView`) - 어떤 기기를 포함할지
3. **기기별 설정** (`QuickIoT DeviceMainView`, `QuickIoT DeviceModeView`) - 각 기기에 어떤 설정을 적용할지

```dart
// QuickAddController
class QuickAddController extends GetxController {
  final RxString name = ''.obs;
  final RxString iconName = 'ic_quick_home'.obs;
  final RxList<QuickModeDevice> selectedDevices = <QuickModeDevice>[].obs;

  Future<void> saveQuickMode() async {
    if (name.value.isEmpty) {
      Get.snackbar('오류', '이름을 입력해주세요');
      return;
    }
    if (selectedDevices.isEmpty) {
      Get.snackbar('오류', '기기를 하나 이상 선택해주세요');
      return;
    }

    final quickMode = QuickMode(
      id: const Uuid().v4(),
      name: name.value,
      iconName: iconName.value,
      devices: selectedDevices.toList(),
      sortOrder: await _getNextSortOrder(),
    );

    final result = await _apiRepo.addQuickMode(quickMode);
    result.fold(
      onSuccess: (_) => Get.until((route) => route.settings.name == Routes.mainTab),
      onFailure: (msg) => Get.snackbar('저장 실패', msg),
    );
  }
}
```

## 퀵 모드 실행

버튼 하나로 여러 기기에 명령을 동시에 보낸다:

```dart
Future<void> executeQuickMode(QuickMode quickMode) async {
  isLoading.value = true;

  // 각 기기에 병렬로 명령 발행
  final futures = quickMode.devices.map((device) async {
    try {
      if (device.targetTemp != null) {
        await _mqttRepo.publishTemperatureControl(
          device.deviceId,
          device.targetTemp!,
        );
      }
      await _mqttRepo.publishModeControl(device.deviceId, device.mode);
    } catch (e) {
      // 개별 기기 실패는 전체를 막지 않음
      KdLog.e('퀵 모드 실행 실패 (${device.deviceId}): $e');
    }
  });

  await Future.wait(futures);
  isLoading.value = false;
  
  Get.snackbar('완료', '${quickMode.name} 모드가 적용됐습니다');
}
```

## 순서 변경 (드래그 앤 드롭)

퀵 모드 순서를 드래그로 변경하는 기능을 `QuickOrderView`에서 구현했다:

```dart
ReorderableListView.builder(
  itemCount: controller.quickModes.length,
  onReorder: controller.reorderQuickModes,
  itemBuilder: (context, index) {
    final mode = controller.quickModes[index];
    return ListTile(
      key: ValueKey(mode.id),
      leading: Icon(iconFromName(mode.iconName)),
      title: Text(mode.name),
      trailing: const Icon(Icons.drag_handle),
    );
  },
)
```

```dart
void reorderQuickModes(int oldIndex, int newIndex) {
  if (newIndex > oldIndex) newIndex--;
  
  final modes = quickModes.toList();
  final item = modes.removeAt(oldIndex);
  modes.insert(newIndex, item);
  quickModes.value = modes;
  
  // 서버에 순서 저장
  _saveOrder(modes.map((m) => m.id).toList());
}
```

---

다음 편은 자동화 설정 화면 구현이다.
