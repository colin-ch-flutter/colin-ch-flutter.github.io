---
layout: post
title: "IoT 모니터링 대시보드 기기 제어 화면 구현"
description: " "
date: 2026-06-07
tags: [Flutter, IoT 모니터링 대시보드, UI, IoT 기기제어, GetX]
comments: true
share: true
---
![IoT 기기 제어](https://images.unsplash.com/photo-1585771724684-38269d6639fd?w=800&q=80)

## IoT 모니터링 대시보드 제어 화면의 복잡도

일반 기기 제어 화면보다 훨씬 복잡하다. 여러 존(Zone), 알람 목록, 에너지 데이터, 운전 이력 등 많은 정보를 한 화면에서 제공해야 한다.

탭 구조로 화면을 나눴다:
- **제어 탭**: 메인 상태 + 존별 제어
- **알람 탭**: 현재 알람 목록
- **설정 탭**: 기기 설정

## IoT DeviceScadaControlView 구조

```dart
class IoT DeviceScadaControlView extends GetView<IoT DeviceScadaControlController> {
  const IoT DeviceScadaControlView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Obx(() => Text(controller.deviceName.value)),
        actions: [
          IconButton(
            icon: const Icon(Icons.settings),
            onPressed: () => Get.toNamed(Routes.iot_deviceScadaSetting),
          ),
        ],
      ),
      body: Obx(() {
        final status = controller.status.value;
        if (status == null) return const Center(child: CircularProgressIndicator());

        return Column(
          children: [
            // 전체 시스템 상태 배너
            _SystemStatusBanner(status: status),
            // 알람이 있으면 알람 배너
            if (status.alarms.isNotEmpty)
              _AlarmBanner(
                alarms: status.alarms,
                onTap: () => _showAlarmList(status.alarms),
              ),
            // 존별 제어
            Expanded(
              child: _ZoneControlList(
                zones: status.zones,
                onZoneControl: controller.controlZone,
              ),
            ),
          ],
        );
      }),
      bottomNavigationBar: _BottomNavBar(
        onScheduleTap: () => Get.toNamed(Routes.iot_deviceScadaSchedule),
        onHistoryTap: () => Get.toNamed(Routes.iot_deviceScadaHistory),
        onEmsTap: () => Get.toNamed(Routes.iot_deviceScadaEms),
      ),
    );
  }
}
```

## 존별 제어 위젯

```dart
class _ZoneControlList extends StatelessWidget {
  final List<ScadaZone> zones;
  final Function(int zoneId, ZoneControlRequest) onZoneControl;

  const _ZoneControlList({required this.zones, required this.onZoneControl});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: zones.length,
      itemBuilder: (context, index) {
        final zone = zones[index];
        return _ZoneCard(
          zone: zone,
          onToggle: (isActive) => onZoneControl(
            zone.id,
            ZoneControlRequest(isActive: isActive),
          ),
          onTempChange: (temp) => onZoneControl(
            zone.id,
            ZoneControlRequest(setpoint: temp),
          ),
        );
      },
    );
  }
}

class _ZoneCard extends StatelessWidget {
  final ScadaZone zone;
  final ValueChanged<bool> onToggle;
  final ValueChanged<int> onTempChange;

  const _ZoneCard({
    required this.zone,
    required this.onToggle,
    required this.onTempChange,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(zone.name, style: Theme.of(context).textTheme.titleMedium),
                  const SizedBox(height: 4),
                  Text(
                    '현재 ${zone.currentTemp.toStringAsFixed(1)}°C | 설정 ${zone.setpoint}°C',
                    style: Theme.of(context).textTheme.bodySmall,
                  ),
                ],
              ),
            ),
            // 온도 조절 버튼
            _TempAdjustButtons(
              value: zone.setpoint,
              enabled: zone.isActive,
              onChanged: onTempChange,
            ),
            const SizedBox(width: 12),
            // 활성화 스위치
            Switch(
              value: zone.isActive,
              onChanged: onToggle,
            ),
          ],
        ),
      ),
    );
  }
}
```

## 알람 처리

IoT 모니터링 대시보드 시스템에서 알람은 중요하다. 경고(Warning)와 오류(Error)를 시각적으로 구분해서 표시한다:

```dart
class _AlarmBanner extends StatelessWidget {
  final List<ScadaAlarm> alarms;
  final VoidCallback onTap;

  const _AlarmBanner({required this.alarms, required this.onTap});

  @override
  Widget build(BuildContext context) {
    final hasError = alarms.any((a) => a.severity == AlarmSeverity.error);
    
    return GestureDetector(
      onTap: onTap,
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        color: hasError ? Colors.red.shade50 : Colors.orange.shade50,
        child: Row(
          children: [
            Icon(
              Icons.warning_amber,
              color: hasError ? Colors.red : Colors.orange,
              size: 20,
            ),
            const SizedBox(width: 8),
            Expanded(
              child: Text(
                '${alarms.length}개의 알람이 있습니다',
                style: TextStyle(
                  color: hasError ? Colors.red : Colors.orange,
                  fontWeight: FontWeight.w500,
                ),
              ),
            ),
            const Icon(Icons.chevron_right, size: 20),
          ],
        ),
      ),
    );
  }
}
```

---

다음 편은 FOTA(무선 펌웨어 업데이트) 구현이다.
