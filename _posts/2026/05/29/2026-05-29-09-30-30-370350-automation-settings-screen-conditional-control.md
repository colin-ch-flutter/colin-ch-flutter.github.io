---
layout: post
title: "자동화 설정 화면 구현 - 조건에 따른 자동 제어"
description: " "
date: 2026-05-29
tags: [Flutter, 자동화, 스마트홈, IoT, UI]
comments: true
share: true
---
![스마트홈 자동화](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## 자동화가 스마트홈을 진짜 '스마트'하게 만든다

퀵 모드는 사용자가 직접 실행해야 한다. 자동화는 조건이 충족되면 자동으로 실행된다. 진정한 의미의 스마트홈 자동화다.

예를 들면:
- "오전 6시 30분이 되면 난방 켜기"
- "외출 감지되면 외출 모드로 전환"
- "기온이 5도 이하로 내려가면 IoT 기기 최소 온도 유지"

## 자동화 데이터 구조

```dart
// domain/models/automation.dart
class Automation {
  final String id;
  final String name;
  final AutomationTrigger trigger;
  final List<AutomationAction> actions;
  final bool isEnabled;

  const Automation({
    required this.id,
    required this.name,
    required this.trigger,
    required this.actions,
    required this.isEnabled,
  });
}

// 트리거: 어떤 조건에서 실행할지
sealed class AutomationTrigger {}

class TimeBasedTrigger extends AutomationTrigger {
  final TimeOfDay time;
  final List<bool> repeatDays;
  TimeBasedTrigger({required this.time, required this.repeatDays});
}

class TemperatureBasedTrigger extends AutomationTrigger {
  final double thresholdTemp;
  final ComparisonOperator operator; // GreaterThan, LessThan
  TemperatureBasedTrigger({required this.thresholdTemp, required this.operator});
}

// 액션: 무엇을 할지
class AutomationAction {
  final String deviceId;
  final ActionType type;
  final Map<String, dynamic> params;
  
  const AutomationAction({
    required this.deviceId,
    required this.type,
    required this.params,
  });
}
```

## 자동화 목록 화면

```dart
class AutomationMainView extends GetView<AutomationMainController> {
  const AutomationMainView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('자동화'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add),
            onPressed: controller.navigateToAdd,
          ),
        ],
      ),
      body: Obx(() {
        if (controller.automations.isEmpty) {
          return _buildEmptyState();
        }

        return ListView.separated(
          itemCount: controller.automations.length,
          separatorBuilder: (_, __) => const Divider(height: 1),
          itemBuilder: (context, index) {
            final automation = controller.automations[index];
            return _AutomationTile(
              automation: automation,
              onToggle: () => controller.toggleAutomation(automation.id),
              onTap: () => controller.navigateToDetail(automation),
            );
          },
        );
      }),
    );
  }

  Widget _buildEmptyState() {
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const Icon(Icons.auto_awesome, size: 64, color: Colors.grey),
          const SizedBox(height: 16),
          const Text('아직 자동화가 없습니다'),
          const SizedBox(height: 8),
          const Text(
            '조건에 따라 기기를 자동으로 제어해보세요',
            style: TextStyle(color: Colors.grey),
            textAlign: TextAlign.center,
          ),
          const SizedBox(height: 24),
          ElevatedButton(
            onPressed: controller.navigateToAdd,
            child: const Text('자동화 추가'),
          ),
        ],
      ),
    );
  }
}
```

## 자동화 카드 표시

```dart
class _AutomationTile extends StatelessWidget {
  final Automation automation;
  final VoidCallback onToggle;
  final VoidCallback onTap;

  const _AutomationTile({
    required this.automation,
    required this.onToggle,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Container(
        width: 48,
        height: 48,
        decoration: BoxDecoration(
          color: automation.isEnabled
              ? Theme.of(context).colorScheme.primaryContainer
              : Colors.grey.shade200,
          borderRadius: BorderRadius.circular(12),
        ),
        child: Icon(
          _triggerIcon(automation.trigger),
          color: automation.isEnabled
              ? Theme.of(context).colorScheme.primary
              : Colors.grey,
        ),
      ),
      title: Text(automation.name),
      subtitle: Text(
        _triggerDescription(automation.trigger),
        style: const TextStyle(fontSize: 12),
      ),
      trailing: Switch(
        value: automation.isEnabled,
        onChanged: (_) => onToggle(),
      ),
      onTap: onTap,
    );
  }

  IconData _triggerIcon(AutomationTrigger trigger) {
    return switch (trigger) {
      TimeBasedTrigger() => Icons.access_time,
      TemperatureBasedTrigger() => Icons.thermostat,
      _ => Icons.bolt,
    };
  }

  String _triggerDescription(AutomationTrigger trigger) {
    return switch (trigger) {
      TimeBasedTrigger(:final time, :final repeatDays) =>
        '${time.format(Get.context!)} - ${_formatDays(repeatDays)}',
      TemperatureBasedTrigger(:final thresholdTemp, :final operator) =>
        '온도가 $thresholdTemp°C ${operator == ComparisonOperator.lessThan ? "이하" : "이상"}일 때',
      _ => '',
    };
  }
}
```

## 서버에서 자동화 실행

자동화 실행 자체는 서버(AWS Lambda 등)에서 담당한다. 앱은 자동화를 정의하고 저장하는 역할만 한다. 서버가 조건을 모니터링하다가 조건이 충족되면 MQTT로 기기에 명령을 보낸다.

앱에서는 자동화가 실행됐을 때 푸시 알림으로 알 수 있도록 FCM 설정이 필요하다.

---

다음 편부터는 알림 시리즈다. FCM 완전 정복부터 시작한다.
