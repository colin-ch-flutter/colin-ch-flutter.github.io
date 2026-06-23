---
layout: post
title: "히스토리 데이터 조회 - 페이징과 날짜 필터링"
description: " "
date: 2026-06-10
tags: [Flutter, 페이징, 히스토리, IoT 모니터링 대시보드, 무한스크롤]
comments: true
share: true
---

# 히스토리 데이터 조회 - 페이징과 날짜 필터링

![데이터 히스토리](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

## 히스토리 기능 요구사항

IoT 모니터링 대시보드 IoT 기기의 운전 이력을 조회하는 화면이다:
- 날짜 범위 필터 (오늘, 최근 7일, 최근 30일, 직접 선택)
- 무한 스크롤 (페이징)
- 각 항목: 시간, 이벤트 타입, 상세 내용
- 로딩 중 스켈레톤 UI

## 날짜 범위 필터

```dart
enum HistoryDateRange {
  today,
  lastWeek,
  lastMonth,
  custom,
}

class IoT DeviceScadaHistoryController extends GetxController {
  final Rx<HistoryDateRange> selectedRange = HistoryDateRange.today.obs;
  final Rx<DateTime> customStart = DateTime.now().obs;
  final Rx<DateTime> customEnd = DateTime.now().obs;
  
  final RxList<HistoryItem> items = <HistoryItem>[].obs;
  final RxBool isLoading = false.obs;
  final RxBool hasMore = true.obs;
  
  int _currentPage = 1;
  static const _pageSize = 20;

  @override
  void onInit() {
    super.onInit();
    loadHistory(refresh: true);
  }

  DateTimeRange get currentDateRange {
    final now = DateTime.now();
    return switch (selectedRange.value) {
      HistoryDateRange.today => DateTimeRange(
          start: DateTime(now.year, now.month, now.day),
          end: now,
        ),
      HistoryDateRange.lastWeek => DateTimeRange(
          start: now.subtract(const Duration(days: 7)),
          end: now,
        ),
      HistoryDateRange.lastMonth => DateTimeRange(
          start: now.subtract(const Duration(days: 30)),
          end: now,
        ),
      HistoryDateRange.custom => DateTimeRange(
          start: customStart.value,
          end: customEnd.value,
        ),
    };
  }

  void changeRange(HistoryDateRange range) {
    selectedRange.value = range;
    loadHistory(refresh: true);
  }

  Future<void> loadHistory({bool refresh = false}) async {
    if (refresh) {
      _currentPage = 1;
      items.clear();
      hasMore.value = true;
    }
    
    if (!hasMore.value || isLoading.value) return;
    
    isLoading.value = true;
    final range = currentDateRange;
    
    final result = await _apiRepo.getScadaHistory(
      deviceId: deviceId,
      from: range.start,
      to: range.end,
      page: _currentPage,
      pageSize: _pageSize,
    );
    
    isLoading.value = false;
    
    result.fold(
      onSuccess: (newItems) {
        items.addAll(newItems);
        _currentPage++;
        hasMore.value = newItems.length == _pageSize;
      },
      onFailure: (msg) => Get.snackbar('오류', msg),
    );
  }
}
```

## 무한 스크롤 구현

`ScrollController`로 스크롤 끝 감지:

```dart
class IoT DeviceScadaHistoryView extends GetView<IoT DeviceScadaHistoryController> {
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(() {
      if (_scrollController.position.pixels >=
          _scrollController.position.maxScrollExtent - 200) {
        // 끝에서 200px 남았을 때 다음 페이지 로드
        controller.loadHistory();
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('운전 이력')),
      body: Column(
        children: [
          // 날짜 필터
          _DateRangeFilter(
            selected: controller.selectedRange.value,
            onChanged: controller.changeRange,
          ),
          // 목록
          Expanded(
            child: Obx(() {
              if (controller.items.isEmpty && controller.isLoading.value) {
                return const _SkeletonList();
              }
              
              if (controller.items.isEmpty) {
                return const _EmptyHistoryView();
              }
              
              return ListView.builder(
                controller: _scrollController,
                itemCount: controller.items.length + (controller.hasMore.value ? 1 : 0),
                itemBuilder: (context, index) {
                  if (index == controller.items.length) {
                    // 마지막 아이템 → 로딩 인디케이터
                    return const Padding(
                      padding: EdgeInsets.all(16),
                      child: Center(child: CircularProgressIndicator()),
                    );
                  }
                  return _HistoryItemTile(item: controller.items[index]);
                },
              );
            }),
          ),
        ],
      ),
    );
  }
}
```

## 스켈레톤 UI

데이터 로딩 중 실제 레이아웃과 비슷한 빈 UI를 보여준다:

```dart
class _SkeletonList extends StatelessWidget {
  const _SkeletonList();

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: 10,
      itemBuilder: (context, index) {
        return const _SkeletonTile();
      },
    );
  }
}

class _SkeletonTile extends StatelessWidget {
  const _SkeletonTile();

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Row(
        children: [
          Container(
            width: 40,
            height: 40,
            decoration: BoxDecoration(
              color: Colors.grey.shade300,
              shape: BoxShape.circle,
            ),
          ),
          const SizedBox(width: 12),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Container(
                  height: 14,
                  width: double.infinity,
                  color: Colors.grey.shade300,
                ),
                const SizedBox(height: 6),
                Container(
                  height: 12,
                  width: 100,
                  color: Colors.grey.shade200,
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

다음 편은 Firebase Remote Config 활용이다.
