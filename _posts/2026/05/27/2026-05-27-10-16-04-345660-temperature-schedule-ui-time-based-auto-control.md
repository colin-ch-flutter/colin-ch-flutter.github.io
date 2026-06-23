---
layout: post
title: "온도 스케줄 설정 UI - 시간대별 온도 자동 조절"
description: " "
date: 2026-05-27
tags: [Flutter, UI, 스케줄, 예약, IoT 기기]
comments: true
share: true
---

# 온도 스케줄 설정 UI - 시간대별 온도 자동 조절

![일정 캘린더](https://images.unsplash.com/photo-1508739773434-c26b3d09e071?w=800&q=80)

## 스케줄 기능의 실제 가치

IoT 기기 스케줄은 생각보다 자주 쓰이는 기능이다. 매일 오전 7시에 난방을 켜두면 출근 준비할 때 집이 따뜻하고, 퇴근 전 30분에 미리 예열하면 집에 도착했을 때 이미 따뜻하다.

이런 시나리오를 지원하는 스케줄 UI를 구현했다.

## 스케줄 데이터 구조

```dart
// domain/models/iot_device_schedule.dart
class IoT DeviceSchedule {
  final String id;
  final String deviceId;
  final int targetTemp;
  final TimeOfDay startTime;
  final List<bool> repeatDays; // 월~일 7개
  final bool isEnabled;
  final IoT DeviceMode mode;

  const IoT DeviceSchedule({
    required this.id,
    required this.deviceId,
    required this.targetTemp,
    required this.startTime,
    required this.repeatDays,
    required this.isEnabled,
    required this.mode,
  });
}
```

## 스케줄 목록 화면

```dart
class IoT DeviceScheduleView extends GetView<IoT DeviceScheduleController> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('스케줄'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add),
            onPressed: () => Get.toNamed(Routes.iot_deviceAddSchedule),
          ),
        ],
      ),
      body: Obx(() {
        final schedules = controller.schedules;
        
        if (schedules.isEmpty) {
          return _EmptyScheduleView();
        }
        
        return ListView.builder(
          itemCount: schedules.length,
          itemBuilder: (context, index) {
            return _ScheduleCard(
              schedule: schedules[index],
              onToggle: controller.toggleSchedule,
              onDelete: controller.deleteSchedule,
              onTap: () => Get.toNamed(
                Routes.iot_deviceAddSchedule,
                arguments: schedules[index],
              ),
            );
          },
        );
      }),
    );
  }
}
```

## 스케줄 추가 화면

```dart
class IoT DeviceAddScheduleView extends GetView<IoT DeviceAddScheduleController> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(controller.isEdit ? '스케줄 수정' : '스케줄 추가')),
      body: Column(
        children: [
          // 시간 선택
          _TimePicker(
            value: controller.selectedTime,
            onChanged: controller.setTime,
          ),
          // 요일 선택
          _DaySelector(
            selected: controller.selectedDays,
            onDayToggled: controller.toggleDay,
          ),
          // 온도 설정
          _TemperatureSlider(
            value: controller.targetTemp,
            onChanged: controller.setTemperature,
          ),
          // 운전 모드 선택
          _ModeSelector(
            value: controller.selectedMode,
            onChanged: controller.setMode,
          ),
        ],
      ),
      bottomNavigationBar: _SaveButton(onPressed: controller.save),
    );
  }
}
```

## 요일 선택 위젯

```dart
class _DaySelector extends StatelessWidget {
  final List<bool> selected;
  final ValueChanged<int> onDayToggled;

  const _DaySelector({required this.selected, required this.onDayToggled});

  @override
  Widget build(BuildContext context) {
    const days = ['월', '화', '수', '목', '금', '토', '일'];
    
    return Obx(() {
      return Row(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: List.generate(7, (index) {
          final isSelected = selected[index];
          return GestureDetector(
            onTap: () => onDayToggled(index),
            child: Container(
              width: 40,
              height: 40,
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                color: isSelected
                    ? Theme.of(context).colorScheme.primary
                    : Colors.transparent,
                border: Border.all(
                  color: isSelected
                      ? Theme.of(context).colorScheme.primary
                      : Colors.grey,
                ),
              ),
              child: Center(
                child: Text(
                  days[index],
                  style: TextStyle(
                    color: isSelected ? Colors.white : Colors.grey,
                    fontWeight: FontWeight.w500,
                  ),
                ),
              ),
            ),
          );
        }),
      );
    });
  }
}
```

## 스케줄 표시 포맷

```dart
String formatSchedule(IoT DeviceSchedule schedule) {
  final time = schedule.startTime;
  final timeStr = '${time.hour.toString().padLeft(2, '0')}:${time.minute.toString().padLeft(2, '0')}';
  
  final days = schedule.repeatDays;
  final dayNames = ['월', '화', '수', '목', '금', '토', '일'];
  
  // 매일
  if (days.every((d) => d)) return '$timeStr 매일';
  
  // 평일
  if (days.take(5).every((d) => d) && !days[5] && !days[6]) {
    return '$timeStr 평일';
  }
  
  // 주말
  if (!days.take(5).any((d) => d) && days[5] && days[6]) {
    return '$timeStr 주말';
  }
  
  // 그 외
  final selectedDays = days.asMap().entries
      .where((e) => e.value)
      .map((e) => dayNames[e.key])
      .join(', ');
  return '$timeStr $selectedDays';
}
```

---

다음 편은 퀵 모드 기능 설계와 구현이다.
