---
layout: post
title: "글로벌 서비스를 위한 타임존 처리 전략"
description: " "
date: 2026-06-12
tags: [Flutter, 타임존, timezone, 글로벌, IoT]
comments: true
share: true
---
![세계 타임존](https://images.unsplash.com/photo-1526378800651-2e3b8c7cb34b?w=800&q=80)

## 타임존이 왜 중요한가

"오전 7시에 난방 켜기" 스케줄을 설정했다고 하자. 이게 한국 시간 7시인가, 서버(UTC) 시간 7시인가, 아니면 기기가 설치된 현지 시간 7시인가?

글로벌 서비스라 다양한 시간대를 지원해야 한다. 같은 서버에서 관리하는 기기들이 각각 다른 시간대에 있을 수 있다.

스케줄은 **기기가 설치된 현지 시간**을 기준으로 동작해야 한다. 이를 위해 기기마다 타임존을 설정하는 기능이 필요하다.

## 사용하는 패키지들

```yaml
dependencies:
  timezone: ^0.10.1
  flutter_timezone: ^5.0.2
  timezone_to_country: ^3.1.0
```

- `timezone`: IANA 타임존 데이터베이스
- `flutter_timezone`: 기기의 현재 타임존 감지
- `timezone_to_country`: 타임존 코드로 국가 정보 조회

## 타임존 초기화

앱 시작 시 타임존 데이터를 초기화한다:

```dart
import 'package:timezone/data/latest.dart' as tz;
import 'package:timezone/timezone.dart' as tz;
import 'package:flutter_timezone/flutter_timezone.dart';

Future<void> initializeTimezone() async {
  tz.initializeTimeZones();
  
  final localTimezone = await FlutterTimezone.getLocalTimezone();
  try {
    tz.setLocalLocation(tz.getLocation(localTimezone));
  } catch (e) {
    // 알 수 없는 타임존이면 UTC 사용
    tz.setLocalLocation(tz.UTC);
  }
}
```

## 기기 타임존 설정 화면

기기를 등록할 때 타임존을 선택하는 화면이 있다. `DeviceConnectTimezoneView`와 `DeviceTimezoneSetupView`가 이 역할을 한다.

```dart
// core/utils/timezone_catalog.dart
class TimezoneCatalog {
  static final List<TimezoneInfo> allTimezones = _buildTimezoneList();

  static List<TimezoneInfo> _buildTimezoneList() {
    return tz.timeZoneDatabase.locations.entries.map((entry) {
      final location = entry.value;
      final offset = location.currentTimeZone.offset;
      final offsetHours = offset ~/ 3600000;
      final offsetMinutes = (offset.abs() % 3600000) ~/ 60000;
      
      final offsetStr = offsetHours >= 0
          ? 'UTC+${offsetHours.toString().padLeft(2, '0')}:${offsetMinutes.toString().padLeft(2, '0')}'
          : 'UTC-${offsetHours.abs().toString().padLeft(2, '0')}:${offsetMinutes.toString().padLeft(2, '0')}';

      return TimezoneInfo(
        id: entry.key,
        displayName: '($offsetStr) ${entry.key.replaceAll('_', ' ')}',
        offset: offset,
      );
    }).toList()
      ..sort((a, b) => a.offset.compareTo(b.offset));
  }

  static TimezoneInfo? getLocalTimezone() {
    final local = tz.local.name;
    return allTimezones.firstWhereOrNull((tz) => tz.id == local);
  }
}
```

## 타임존 선택 UI

타임존 목록은 상당히 길다. 검색 기능을 추가하고, 현재 기기 타임존을 기본 선택값으로 사용한다:

```dart
class DeviceTimezoneSetupController extends GetxController {
  final RxList<TimezoneInfo> filteredTimezones = 
      TimezoneCatalog.allTimezones.obs;
  final Rx<TimezoneInfo?> selected = Rx(null);
  final TextEditingController searchController = TextEditingController();

  @override
  void onInit() {
    super.onInit();
    // 기기 타임존으로 초기 선택
    selected.value = TimezoneCatalog.getLocalTimezone();
    
    searchController.addListener(_onSearchChanged);
  }

  void _onSearchChanged() {
    final query = searchController.text.toLowerCase();
    if (query.isEmpty) {
      filteredTimezones.value = TimezoneCatalog.allTimezones;
      return;
    }
    filteredTimezones.value = TimezoneCatalog.allTimezones
        .where((tz) => tz.displayName.toLowerCase().contains(query))
        .toList();
  }
}
```

## 서버에 타임존 저장

기기 등록 시 타임존을 서버에 전달한다:

```dart
// IANA 타임존 ID로 저장 (예: 'Asia/Seoul', 'Europe/Moscow')
await _apiRepo.updateDeviceTimezone(
  deviceId: deviceId,
  timezone: selected.value!.id,
);
```

서버에서는 이 타임존을 기준으로 스케줄을 계산한다.

## 타임존 변환 유틸리티

날짜/시간을 표시할 때 기기의 타임존으로 변환해서 보여준다:

```dart
extension DateTimeExtension on DateTime {
  String toDeviceLocalTime(String deviceTimezoneId) {
    try {
      final location = tz.getLocation(deviceTimezoneId);
      final localDt = tz.TZDateTime.from(this, location);
      return DateFormat('yyyy-MM-dd HH:mm').format(localDt);
    } catch (_) {
      return DateFormat('yyyy-MM-dd HH:mm').format(toLocal());
    }
  }
}
```

---

다음 편은 Flutter 앱 성능 최적화다.
