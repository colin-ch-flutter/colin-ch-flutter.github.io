---
layout: post
title: "Repository 패턴으로 깔끔한 데이터 레이어 구성"
description: " "
date: 2026-05-19
tags: [Flutter, Repository패턴, CleanArchitecture, DI, GetX]
comments: true
share: true
---

# Repository 패턴으로 깔끔한 데이터 레이어 구성

![코드 구조](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## Repository 패턴이 필요한 이유

Controller에서 직접 API를 호출하거나, MQTT 클라이언트를 직접 써도 동작은 한다. 그런데 이렇게 하면:

1. **테스트 어렵다**: 실제 서버 없이 테스트 불가
2. **교체 어렵다**: API 라이브러리를 바꾸려면 Controller를 전부 수정해야 함
3. **중복 코드**: 같은 API 호출이 여러 Controller에 중복됨

Repository 패턴은 데이터 소스를 추상화해서 이 문제를 해결한다.

## 인터페이스 정의 (Domain)

Domain 레이어에 인터페이스만 정의한다:

```dart
// domain/repositories/api_repository.dart
abstract class ApiRepository {
  Future<ApiResult<List<IoT DeviceDevice>>> getDevices(String spaceId);
  Future<ApiResult<IoT DeviceDevice>> addDevice(AddDeviceRequest request);
  Future<ApiResult<void>> deleteDevice(String deviceId);
  Future<ApiResult<void>> updateDeviceNickname(String deviceId, String nickname);
}
```

`ApiResult`는 성공/실패를 타입 안전하게 표현하는 래퍼다:

```dart
// domain/common/api_result.dart
sealed class ApiResult<T> {
  const ApiResult();
}

class ApiSuccess<T> extends ApiResult<T> {
  final T data;
  const ApiSuccess(this.data);
}

class ApiFailure<T> extends ApiResult<T> {
  final String message;
  final int? statusCode;
  const ApiFailure(this.message, {this.statusCode});
}
```

## 구현체 (Data)

```dart
// data/repositories/api_repository_impl.dart
class ApiRepositoryImpl implements ApiRepository {
  final ApiService _apiService;

  ApiRepositoryImpl(this._apiService);

  @override
  Future<ApiResult<List<IoT DeviceDevice>>> getDevices(String spaceId) async {
    try {
      final response = await _apiService.get('/spaces/$spaceId/devices');
      final devices = (response.data as List)
          .map((json) => IoT DeviceDeviceApiMapper.fromJson(json))
          .toList();
      return ApiSuccess(devices);
    } on ApiException catch (e) {
      return ApiFailure(e.message, statusCode: e.statusCode);
    } catch (e) {
      return ApiFailure('알 수 없는 오류가 발생했습니다');
    }
  }
}
```

## Controller에서 사용

```dart
class DeviceManagementController extends GetxController {
  final ApiRepository _apiRepo;
  
  final RxList<IoT DeviceDevice> devices = <IoT DeviceDevice>[].obs;
  final RxBool isLoading = false.obs;
  final RxString errorMessage = ''.obs;

  DeviceManagementController(this._apiRepo);

  Future<void> loadDevices(String spaceId) async {
    isLoading.value = true;
    errorMessage.value = '';
    
    final result = await _apiRepo.getDevices(spaceId);
    
    switch (result) {
      case ApiSuccess(:final data):
        devices.value = data;
      case ApiFailure(:final message):
        errorMessage.value = message;
    }
    
    isLoading.value = false;
  }
}
```

Dart 3의 패턴 매칭으로 `ApiResult`를 깔끔하게 처리할 수 있다.

## Binding으로 DI 구성

GetX Binding에서 Repository를 등록하고 Controller에 주입한다:

```dart
// core/bindings/initial_binding.dart
class InitialBinding extends Bindings {
  @override
  void dependencies() {
    // 서비스 레이어 (싱글톤)
    Get.lazyPut<ApiService>(() => ApiService(), fenix: true);
    Get.lazyPut<IotMqttService>(() => IotMqttService(), fenix: true);
    Get.lazyPut<DbService>(() => DbService(), fenix: true);
    
    // Repository (싱글톤)
    Get.lazyPut<ApiRepository>(
      () => ApiRepositoryImpl(Get.find()),
      fenix: true,
    );
    Get.lazyPut<IoT DeviceMqttRepository>(
      () => IoT DeviceMqttRepositoryImpl(Get.find()),
      fenix: true,
    );
    Get.lazyPut<DbRepository>(
      () => DbRepositoryImpl(Get.find()),
      fenix: true,
    );
  }
}
```

화면별 Binding에서 Controller를 등록:

```dart
class DeviceManagementBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => DeviceManagementController(Get.find<ApiRepository>()));
  }
}
```

## 테스트 모드에서의 교체

실제 기기 없이 개발할 때 `data/test/` 의 Mock 구현체로 교체:

```dart
// 테스트 환경에서
Get.lazyPut<ApiRepository>(
  () => TestApiRepositoryImpl(),  // Mock 구현체
  fenix: true,
);
```

Controller 코드는 전혀 바꾸지 않아도 Mock이 들어간다. 이게 Repository 패턴의 가장 큰 장점이다.

---

다음 편은 JWT 토큰 자동 갱신을 구현하는 방법이다.
