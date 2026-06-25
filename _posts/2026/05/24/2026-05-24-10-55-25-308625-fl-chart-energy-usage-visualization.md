---
layout: post
title: "fl_chart로 에너지 사용량 시각화하기"
description: " "
date: 2026-05-24
tags: [Flutter, fl_chart, 차트, 데이터시각화, IoT]
comments: true
share: true
---
![데이터 차트](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

## fl_chart 선택 이유

Flutter에서 차트 라이브러리는 몇 가지가 있다. `fl_chart`를 선택한 이유는 커스터마이징이 자유롭고, 애니메이션 지원이 좋으며, 문서도 잘 정리되어 있기 때문이다.

## 온도 이력 라인 차트

시간에 따른 온도 변화를 라인 차트로 표시한다:

```dart
LineChart(
  LineChartData(
    lineBarsData: [
      // 현재 온도 라인
      LineChartBarData(
        spots: temperatureHistory.map((point) {
          return FlSpot(
            point.timestamp.millisecondsSinceEpoch.toDouble(),
            point.temperature,
          );
        }).toList(),
        isCurved: true,
        color: Theme.of(context).colorScheme.primary,
        barWidth: 2,
        dotData: const FlDotData(show: false),
        belowBarData: BarAreaData(
          show: true,
          color: Theme.of(context).colorScheme.primary.withOpacity(0.1),
        ),
      ),
    ],
    titlesData: FlTitlesData(
      leftTitles: AxisTitles(
        sideTitles: SideTitles(
          showTitles: true,
          reservedSize: 40,
          getTitlesWidget: (value, meta) {
            return Text(
              '${value.toInt()}°C',
              style: const TextStyle(fontSize: 10),
            );
          },
        ),
      ),
      bottomTitles: AxisTitles(
        sideTitles: SideTitles(
          showTitles: true,
          reservedSize: 30,
          getTitlesWidget: (value, meta) {
            final dt = DateTime.fromMillisecondsSinceEpoch(value.toInt());
            return Text(
              DateFormat('HH:mm').format(dt),
              style: const TextStyle(fontSize: 10),
            );
          },
        ),
      ),
      topTitles: const AxisTitles(sideTitles: SideTitles(showTitles: false)),
      rightTitles: const AxisTitles(sideTitles: SideTitles(showTitles: false)),
    ),
    gridData: FlGridData(
      show: true,
      drawVerticalLine: false,
      horizontalInterval: 5,
      getDrawingHorizontalLine: (value) {
        return FlLine(
          color: Colors.grey.withOpacity(0.2),
          strokeWidth: 1,
        );
      },
    ),
    borderData: FlBorderData(show: false),
    lineTouchData: LineTouchData(
      touchTooltipData: LineTouchTooltipData(
        getTooltipItems: (spots) {
          return spots.map((spot) {
            final dt = DateTime.fromMillisecondsSinceEpoch(spot.x.toInt());
            return LineTooltipItem(
              '${DateFormat('HH:mm').format(dt)}\n${spot.y.toStringAsFixed(1)}°C',
              const TextStyle(color: Colors.white, fontSize: 12),
            );
          }).toList();
        },
      ),
    ),
  ),
)
```

## 일별 가스 사용량 바 차트

```dart
BarChart(
  BarChartData(
    barGroups: weeklyUsage.asMap().entries.map((entry) {
      return BarChartGroupData(
        x: entry.key,
        barRods: [
          BarChartRodData(
            toY: entry.value.gasUsage,
            color: Theme.of(context).colorScheme.primary,
            width: 20,
            borderRadius: const BorderRadius.vertical(top: Radius.circular(4)),
          ),
        ],
      );
    }).toList(),
    titlesData: FlTitlesData(
      bottomTitles: AxisTitles(
        sideTitles: SideTitles(
          showTitles: true,
          getTitlesWidget: (value, meta) {
            const days = ['월', '화', '수', '목', '금', '토', '일'];
            return Text(
              days[value.toInt() % 7],
              style: const TextStyle(fontSize: 12),
            );
          },
        ),
      ),
      leftTitles: AxisTitles(
        sideTitles: SideTitles(
          showTitles: true,
          reservedSize: 40,
          getTitlesWidget: (value, meta) {
            return Text('${value.toInt()}m³', style: const TextStyle(fontSize: 10));
          },
        ),
      ),
      topTitles: const AxisTitles(sideTitles: SideTitles(showTitles: false)),
      rightTitles: const AxisTitles(sideTitles: SideTitles(showTitles: false)),
    ),
    barTouchData: BarTouchData(
      touchTooltipData: BarTouchTooltipData(
        getTooltipItem: (group, groupIndex, rod, rodIndex) {
          return BarTooltipItem(
            '${rod.toY.toStringAsFixed(2)}m³',
            const TextStyle(color: Colors.white),
          );
        },
      ),
    ),
  ),
)
```

## 데이터 없을 때 처리

데이터가 없을 때 차트를 그리려고 하면 오류가 발생하거나 이상하게 표시된다:

```dart
Widget buildChart(List<TemperaturePoint> data) {
  if (data.isEmpty) {
    return const Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(Icons.bar_chart, size: 48, color: Colors.grey),
          SizedBox(height: 8),
          Text('데이터가 없습니다', style: TextStyle(color: Colors.grey)),
        ],
      ),
    );
  }
  
  if (data.length == 1) {
    // 점이 하나면 라인 차트가 이상하게 보임 → 다른 표시 방법 사용
    return _SinglePointDisplay(data: data.first);
  }
  
  return _buildLineChart(data);
}
```

---

다음 편은 Lottie 애니메이션으로 앱에 생동감을 불어넣는 방법이다.
