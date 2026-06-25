---
layout: post
title: "오프라인 캐시 전략 - 인터넷 없어도 최후 상태 보여주기"
description: " "
date: 2026-05-22
tags: [Flutter, 오프라인, 캐시, connectivity_plus, IoT]
comments: true
share: true
---
![오프라인 캐시](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## IoT 앱에서 오프라인이 중요한 이유

사용자가 앱을 열었는데 "연결 없음" 에러만 보이면 당황스럽다. 특히 IoT 앱은 집 안에서 열리는 경우가 많은데, 집 WiFi가 잠깐 끊기거나 이동통신 신호가 약한 경우가 있다.

이럴 때 마지막으로 받은 기기 상태라도 보여주는 게 훨씬 나은 UX다.

## 네트워크 상태 감지

```dart
// connectivity_plus로 현재 네트워크 상태 확인
import 'package:connectivity_plus/connectivity_plus.dart';

class NetworkStatusService {
  final Connectivity _connectivity = Connectivity();
  
  final RxBool isOnline = true.obs;
  
  void init() {
    // 현재 상태 확인
    _connectivity.checkConnectivity().then(_updateStatus);
    
    // 변경 감지
    _connectivity.onConnectivityChanged.listen(_updateStatus);
  }
  
  void _updateStatus(ConnectivityResult result) {
    isOnline.value = result != ConnectivityResult.none;
  }
}
```

## 캐시 우선 전략

앱 시작 시 캐시 데이터를 먼저 보여주고, 네트워크 요청이 성공하면 업데이트한다:

```dart
// data/cache/app_cache_impl.dart
class AppCacheImpl implements AppCache {
  final GetStorage _storage = GetStorage();
  
  @override
  Future<void> saveDeviceList(List<IoT DeviceDevice> devices) async {
    final json = devices.map((d) => d.toJson()).toList();
    await _storage.write('cached_devices', json);
    await _storage.write('cached_devices_at', DateTime.now().toIso8601String());
  }
  
  @override
  List<IoT DeviceDevice>? loadDeviceList() {
    final json = _storage.read<List>('cached_devices');
    if (json == null) return null;
    
    return json
        .map((j) => IoT DeviceDevice.fromJson(j as Map<String, dynamic>))
        .toList();
  }
  
  @override
  bool isCacheStale({Duration maxAge = const Duration(hours: 1)}) {
    final cachedAtStr = _storage.read<String>('cached_devices_at');
    if (cachedAtStr == null) return true;
    
    final cachedAt = DateTime.parse(cachedAtStr);
    return DateTime.now().difference(cachedAt) > maxAge;
  }
}
```

## Repository에서 캐시 활용

```dart
class ApiRepositoryImpl implements ApiRepository {
  final ApiService _apiService;
  final AppCache _cache;

  @override
  Future<ApiResult<List<IoT DeviceDevice>>> getDevices(String spaceId) async {
    // 1. 캐시 데이터 먼저 반환 (있을 경우)
    final cached = _cache.loadDeviceList();
    
    // 2. 네트워크 요청 시도
    try {
      final response = await _apiService.get('/spaces/$spaceId/devices')
          .timeout(const Duration(seconds: 10));
      
      final devices = _parseDevices(response);
      _cache.saveDeviceList(devices); // 성공 시 캐시 갱신
      return ApiSuccess(devices);
      
    } catch (e) {
      // 3. 실패 시 캐시 반환
      if (cached != null) {
        return ApiSuccess(cached);  // 캐시 데이터로 성공 반환
      }
      return ApiFailure('기기 목록을 불러올 수 없습니다. 인터넷 연결을 확인해주세요.');
    }
  }
}
```

## MQTT 스냅샷 캐시

MQTT로 받은 기기 상태도 캐시한다:

```dart
// domain/ports/iot_device_snapshot_cache.dart
abstract class IoT DeviceSnapshotCache {
  void save(String deviceId, IoT DeviceStatus status);
  IoT DeviceStatus? load(String deviceId);
}

// 구현
class IoT DeviceSnapshotCacheImpl implements IoT DeviceSnapshotCache {
  final Map<String, IoT DeviceStatus> _cache = {};
  final GetStorage _storage = GetStorage();

  @override
  void save(String deviceId, IoT DeviceStatus status) {
    _cache[deviceId] = status;
    // 앱 재시작 대비 영구 저장
    _storage.write('snapshot_$deviceId', status.toJson());
  }

  @override
  IoT DeviceStatus? load(String deviceId) {
    // 메모리 캐시 먼저
    if (_cache.containsKey(deviceId)) return _cache[deviceId];
    
    // 디스크 캐시
    final json = _storage.read<Map>('snapshot_$deviceId');
    if (json == null) return null;
    
    return IoT DeviceStatus.fromJson(Map<String, dynamic>.from(json));
  }
}
```

## 오프라인 UI 처리

오프라인 상태를 사용자에게 표시하는 방법:

```dart
// 화면 상단에 오프라인 배너
Obx(() {
  if (!networkService.isOnline.value) {
    return Container(
      width: double.infinity,
      color: Colors.orange,
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Text(
        '오프라인 상태 - 마지막으로 저장된 정보를 표시합니다',
        textAlign: TextAlign.center,
        style: const TextStyle(color: Colors.white, fontSize: 12),
      ),
    );
  }
  return const SizedBox.shrink();
})
```

---

다음 편부터는 UI/UX 파트다. 기기 제어 화면 구현부터 시작한다.
