---
layout: post
title: "http 패키지로 REST API 통신 구현 - Dio 없이도 충분하다"
description: " "
date: 2026-05-21
tags: [Flutter, HTTP, REST, API, json_serializable]
comments: true
share: true
---

# http 패키지로 REST API 통신 구현 - Dio 없이도 충분하다

![API 통신](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

## Dio vs http 패키지

Flutter에서 HTTP 통신을 하면 대부분 Dio를 쓴다. 인터셉터, 파일 업로드, 취소 등 기능이 풍부하다. 근데 이 프로젝트에서는 기본 `http` 패키지를 썼다.

이유는 단순하다. Dio가 제공하는 고급 기능이 이 프로젝트에서는 크게 필요하지 않았고, 의존성을 최소화하는 게 장기적으로 유지보수에 유리하다고 판단했다. 인터셉터 기능은 TokenRefresher 클래스로 대체하고, 공통 헤더는 래퍼 클래스로 처리했다.

## ApiService 래퍼 구현

```dart
// data/services/api_service.dart
class ApiService {
  final String baseUrl;
  final TokenRefresher _tokenRefresher;

  ApiService({required this.baseUrl, required TokenRefresher tokenRefresher})
      : _tokenRefresher = tokenRefresher;

  Future<Map<String, String>> _getHeaders() async {
    final token = await _tokenRefresher.getValidAccessToken();
    return {
      'Authorization': 'Bearer $token',
      'Content-Type': 'application/json; charset=utf-8',
      'Accept': 'application/json',
    };
  }

  Future<ApiResponse> get(String path, {Map<String, String>? queryParams}) async {
    final uri = Uri.parse(baseUrl + path).replace(queryParameters: queryParams);
    final headers = await _getHeaders();

    final response = await http.get(uri, headers: headers);
    return _handleResponse(response);
  }

  Future<ApiResponse> post(String path, {Object? body}) async {
    final uri = Uri.parse(baseUrl + path);
    final headers = await _getHeaders();
    final encodedBody = body != null ? jsonEncode(body) : null;

    final response = await http.post(uri, headers: headers, body: encodedBody);
    return _handleResponse(response);
  }

  Future<ApiResponse> put(String path, {Object? body}) async {
    final uri = Uri.parse(baseUrl + path);
    final headers = await _getHeaders();
    final encodedBody = body != null ? jsonEncode(body) : null;

    final response = await http.put(uri, headers: headers, body: encodedBody);
    return _handleResponse(response);
  }

  Future<ApiResponse> delete(String path) async {
    final uri = Uri.parse(baseUrl + path);
    final headers = await _getHeaders();

    final response = await http.delete(uri, headers: headers);
    return _handleResponse(response);
  }

  ApiResponse _handleResponse(http.Response response) {
    // UTF-8 디코딩 (charset_converter 활용)
    final body = utf8.decode(response.bodyBytes);
    
    if (response.statusCode >= 200 && response.statusCode < 300) {
      return ApiResponse.success(
        statusCode: response.statusCode,
        body: body,
      );
    }
    
    throw ApiException(
      statusCode: response.statusCode,
      message: _extractErrorMessage(body),
    );
  }

  String _extractErrorMessage(String body) {
    try {
      final json = jsonDecode(body) as Map<String, dynamic>;
      return json['message'] as String? ?? '오류가 발생했습니다';
    } catch (_) {
      return '오류가 발생했습니다 (${body.length > 100 ? body.substring(0, 100) : body})';
    }
  }
}
```

## json_serializable로 모델 자동 생성

JSON 파싱 코드를 일일이 쓰는 건 귀찮고 오타 실수도 잦다. `json_serializable`을 쓰면 자동 생성된다:

```dart
// data/models/device_model.dart
import 'package:json_annotation/json_annotation.dart';

part 'device_model.g.dart';

@JsonSerializable()
class DeviceApiModel {
  @JsonKey(name: 'device_id')
  final String deviceId;
  
  @JsonKey(name: 'nick_name')
  final String? nickName;
  
  @JsonKey(name: 'model_name')
  final String modelName;
  
  @JsonKey(name: 'is_online')
  final bool isOnline;

  const DeviceApiModel({
    required this.deviceId,
    this.nickName,
    required this.modelName,
    required this.isOnline,
  });

  factory DeviceApiModel.fromJson(Map<String, dynamic> json) =>
      _$DeviceApiModelFromJson(json);
  
  Map<String, dynamic> toJson() => _$DeviceApiModelToJson(this);
}
```

```bash
flutter pub run build_runner build
```

`device_model.g.dart`에 `fromJson`, `toJson`이 자동으로 생성된다.

## ApiRepository 구현체 예시

```dart
@override
Future<ApiResult<List<IoT DeviceDevice>>> getDevices(String spaceId) async {
  try {
    final response = await _apiService.get(
      '/v1/spaces/$spaceId/devices',
    );
    
    final jsonList = jsonDecode(response.body) as List<dynamic>;
    final devices = jsonList
        .map((json) => DeviceApiModel.fromJson(json as Map<String, dynamic>))
        .map(DeviceApiMapper.fromApiModel)
        .toList();
    
    return ApiSuccess(devices);
  } on ApiException catch (e) {
    return ApiFailure(e.message, statusCode: e.statusCode);
  } catch (e) {
    return ApiFailure('기기 목록을 불러오는 데 실패했습니다');
  }
}
```

## 로깅

개발 중 API 요청/응답을 로깅하면 디버깅에 유용하다:

```dart
if (kDebugMode) {
  print('[API] ${method.toUpperCase()} $path');
  if (body != null) print('[REQ] $body');
  print('[RES ${response.statusCode}] $body');
}
```

---

다음 편은 오프라인 캐시 전략이다.
