---
layout: post
title: "EMS(에너지 관리 시스템) 화면 구현"
description: " "
date: 2026-06-09
tags: [Flutter, EMS, 에너지관리, fl_chart, IoT 모니터링 대시보드]
comments: true
share: true
---
![에너지 관리](https://images.unsplash.com/photo-1473341304170-971dccb5ac1e?w=800&q=80)

## EMS가 뭔가

EMS(Energy Management System)는 에너지 사용량을 모니터링하고 효율을 관리하는 시스템이다. 상업용 IoT 기기를 운영하는 건물 관리자나 시설 담당자에게 중요한 기능이다.

IoT 모니터링 대시보드 IoT 기기의 EMS 화면에서 제공하는 정보:
- 실시간 가스 소비량
- 실시간 전력 소비량
- 시간별 / 일별 / 월별 에너지 트렌드
- 에너지 효율 지표 (COP 등)
- 비교 기간 대비 증감율

## EMS Controller

```dart
class IoT DeviceScadaEmsController extends GetxController {
  final String deviceId;
  
  final Rx<EmsData?> currentData = Rx(null);
  final RxList<EmsHistory> hourlyHistory = <EmsHistory>[].obs;
  final Rx<EmsDateRange> selectedRange = EmsDateRange.daily.obs;
  final RxBool isLoading = false.obs;

  @override
  void onInit() {
    super.onInit();
    loadData();
    // 실시간 데이터 구독
    _mqttRepo.emsStream
        .where((d) => d.deviceId == deviceId)
        .listen((data) => currentData.value = data);
  }

  Future<void> loadData() async {
    isLoading.value = true;
    
    final range = _getDateRange(selectedRange.value);
    final result = await _apiRepo.getEmsHistory(
      deviceId: deviceId,
      from: range.start,
      to: range.end,
      granularity: selectedRange.value.granularity,
    );
    
    isLoading.value = false;
    
    result.fold(
      onSuccess: (history) => hourlyHistory.value = history,
      onFailure: (msg) => Get.snackbar('오류', msg),
    );
  }

  void changeRange(EmsDateRange range) {
    selectedRange.value = range;
    loadData();
  }
}
```

## EMS View 구현

```dart
class IoT DeviceScadaEmsView extends GetView<IoT DeviceScadaEmsController> {
  const IoT DeviceScadaEmsView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('에너지 관리')),
      body: Obx(() {
        return SingleChildScrollView(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // 현재 실시간 데이터
              _RealTimeEmsCard(data: controller.currentData.value),
              const SizedBox(height: 16),
              // 기간 선택 탭
              _DateRangeSelector(
                selected: controller.selectedRange.value,
                onChanged: controller.changeRange,
              ),
              const SizedBox(height: 16),
              // 가스 사용량 차트
              _EmsChart(
                title: '가스 사용량 (m³)',
                history: controller.hourlyHistory,
                valueGetter: (h) => h.gasUsage,
                color: Colors.orange,
              ),
              const SizedBox(height: 16),
              // 전력 사용량 차트
              _EmsChart(
                title: '전력 사용량 (kWh)',
                history: controller.hourlyHistory,
                valueGetter: (h) => h.powerUsage,
                color: Colors.blue,
              ),
              const SizedBox(height: 16),
              // 효율 지표
              _EfficiencyCard(history: controller.hourlyHistory),
            ],
          ),
        );
      }),
    );
  }
}
```

## 실시간 데이터 카드

```dart
class _RealTimeEmsCard extends StatelessWidget {
  final EmsData? data;

  const _RealTimeEmsCard({required this.data});

  @override
  Widget build(BuildContext context) {
    if (data == null) {
      return const Card(
        child: Padding(
          padding: EdgeInsets.all(16),
          child: Center(child: Text('데이터 없음')),
        ),
      );
    }

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Expanded(
              child: _EmsIndicator(
                label: '가스',
                value: '${data!.gasUsage.toStringAsFixed(2)} m³/h',
                icon: Icons.local_fire_department,
                color: Colors.orange,
              ),
            ),
            const VerticalDivider(),
            Expanded(
              child: _EmsIndicator(
                label: '전력',
                value: '${data!.powerUsage.toStringAsFixed(1)} kW',
                icon: Icons.bolt,
                color: Colors.blue,
              ),
            ),
            const VerticalDivider(),
            Expanded(
              child: _EmsIndicator(
                label: '효율',
                value: '${data!.efficiency.toStringAsFixed(1)}%',
                icon: Icons.eco,
                color: Colors.green,
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

## fl_chart로 에너지 트렌드

EMS 차트는 시간에 따른 에너지 소비 트렌드를 보여준다. 비교 기간 데이터도 함께 표시하면 개선 여부를 한눈에 볼 수 있다:

```dart
class _EmsChart extends StatelessWidget {
  final String title;
  final List<EmsHistory> history;
  final double Function(EmsHistory) valueGetter;
  final Color color;

  const _EmsChart({
    required this.title,
    required this.history,
    required this.valueGetter,
    required this.color,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(title, style: Theme.of(context).textTheme.titleMedium),
        const SizedBox(height: 8),
        SizedBox(
          height: 200,
          child: BarChart(
            BarChartData(
              barGroups: history.asMap().entries.map((entry) {
                return BarChartGroupData(
                  x: entry.key,
                  barRods: [
                    BarChartRodData(
                      toY: valueGetter(entry.value),
                      color: color,
                      width: 8,
                      borderRadius: const BorderRadius.vertical(
                        top: Radius.circular(2),
                      ),
                    ),
                  ],
                );
              }).toList(),
            ),
          ),
        ),
      ],
    );
  }
}
```

---

다음 편은 IoT 모니터링 대시보드 히스토리 데이터 조회와 페이징 처리다.
