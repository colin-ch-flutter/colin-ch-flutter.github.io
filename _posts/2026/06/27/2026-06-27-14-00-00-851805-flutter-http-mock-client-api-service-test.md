---
layout: post
title: "http.MockClient로 ApiService 단위 테스트 격리 — 서버 없이 401·오류 케이스 재현"
description: "http 패키지 MockClient를 ApiService에 주입해 REST API 레이어를 격리 테스트하는 방법을 정리했다. 200 성공부터 401 토큰 만료, 네트워크 오류까지 실제 서버 없이 재현할 수 있다."
date: 2026-06-27
tags: [Flutter, Dart, HTTP, CleanArchitecture, Repository패턴]
comments: true
share: true
---

![REST API 테스트 격리 전략](https://images.unsplash.com/photo-1526374965328-7f61d4dc18c5?w=800&q=80)

REST API 레이어를 테스트하려고 할 때 처음엔 막막했다. 실제 서버가 올라가야 하나? 테스트용 서버를 따로 띄워야 하나? IoT 앱에서 `http` 패키지를 쓰는 ApiService는 `http.get()` 전역 함수를 직접 호출하고 있어서 가로챌 방법이 없어 보였다. 근데 알고 보면 `package:http/testing.dart`의 `MockClient`를 쓰면 된다. ApiService 생성자에서 `http.Client`를 주입받도록 딱 한 줄 바꾸면, 테스트에서 모든 HTTP 응답을 직접 만들어 줄 수 있다.

## 왜 ApiService 레이어를 격리해야 하나

[JWT 자동갱신 포스트]({% post_url 2026-05-20-09-27-33-259245-jwt-token-auto-refresh-session-management %})에서 TokenRefresher를 구현했다. 근데 실제로 문제가 터지는 건 대부분 ApiService 레이어다. 401이 오면 토큰을 갱신하고 재시도하는 로직, 403·500·네트워크 오류별 에러 핸들링 — 이걸 실제 서버로만 테스트하면 케이스를 만드는 것 자체가 너무 어렵다.

의도적으로 서버에서 401을 반환하게 하려면? 테스트 전용 엔드포인트를 만들거나 토큰을 일부러 만료시키거나... 다 번거롭다. MockClient로 격리하면 `return http.Response('', 401)` 한 줄로 끝난다.

## ApiService를 주입 가능하게 리팩토링

기존 ApiService는 `http.get()` 전역 함수를 직접 호출하고 있었다. 테스트에서 가로채려면 `http.Client` 인스턴스를 외부에서 주입받도록 바꿔야 한다.

```dart
// data/services/api_service.dart
import 'package:http/http.dart' as http;

class ApiService {
  final String baseUrl;
  final TokenRefresher _tokenRefresher;
  final http.Client _client; // 추가

  ApiService({
    required this.baseUrl,
    required TokenRefresher tokenRefresher,
    http.Client? client, // null이면 기본 http.Client() 사용
  })  : _tokenRefresher = tokenRefresher,
        _client = client ?? http.Client();

  Future<ApiResponse> get(String path, {Map<String, String>? queryParams}) async {
    final uri = Uri.parse(baseUrl + path).replace(queryParameters: queryParams);
    final headers = await _getHeaders();

    // http.get() → _client.get() 으로 교체
    final response = await _client.get(uri, headers: headers);
    return _handleResponse(response);
  }

  Future<ApiResponse> post(String path, {Object? body}) async {
    final uri = Uri.parse(baseUrl + path);
    final headers = await _getHeaders();
    final encodedBody = body != null ? jsonEncode(body) : null;

    final response = await _client.post(uri, headers: headers, body: encodedBody);
    return _handleResponse(response);
  }
}
```

프로덕션 코드에서는 `ApiService(baseUrl: ..., tokenRefresher: ...)` 그대로 쓰면 내부적으로 `http.Client()`가 생성된다. 테스트에서만 `client: MockClient(...)` 를 넣어주면 된다. 기존 코드 변경 최소화가 핵심이다.

## MockClient 기본 사용

```dart
import 'package:http/testing.dart';
import 'package:http/http.dart' as http;
import 'package:test/test.dart';

void main() {
  late ApiService apiService;
  late FakeTokenRefresher fakeTokenRefresher;

  setUp(() {
    fakeTokenRefresher = FakeTokenRefresher();
  });

  test('GET 요청 성공 — 200 응답 파싱', () async {
    final mockClient = MockClient((request) async {
      // 요청 URL 검증
      expect(request.url.path, equals('/v1/devices'));
      expect(request.headers['Authorization'], startsWith('Bearer '));

      return http.Response(
        '{"devices": [{"id": "d1", "name": "보일러"}]}',
        200,
        headers: {'content-type': 'application/json; charset=utf-8'},
      );
    });

    apiService = ApiService(
      baseUrl: 'https://api.example.com',
      tokenRefresher: fakeTokenRefresher,
      client: mockClient,
    );

    final response = await apiService.get('/v1/devices');

    expect(response.statusCode, equals(200));
    expect(response.body, contains('"id": "d1"'));
  });
}
```

MockClient 콜백은 `Request`를 받아서 `Response`를 반환한다. URL, 헤더, 바디 전부 검증하거나, 원하는 응답을 자유롭게 만들 수 있다.

## 401 토큰 만료 시나리오

![토큰 갱신 후 재시도 흐름](https://images.unsplash.com/photo-1555949963-aa79dcee981c?w=800&q=80)

이게 핵심 테스트 케이스다. 첫 번째 요청에서 401을 받으면 토큰을 갱신하고 재시도하는 로직이 제대로 작동하는지 확인한다.

```dart
test('401 응답 시 토큰 갱신 후 재시도', () async {
  var callCount = 0;

  final mockClient = MockClient((request) async {
    callCount++;
    if (callCount == 1) {
      // 첫 번째 요청 — 토큰 만료
      return http.Response('{"error": "unauthorized"}', 401);
    }
    // 두 번째 요청 — 갱신된 토큰으로 재시도
    expect(request.headers['Authorization'], equals('Bearer new-token'));
    return http.Response(
      '{"devices": []}',
      200,
      headers: {'content-type': 'application/json'},
    );
  });

  fakeTokenRefresher.nextToken = 'new-token'; // 갱신 후 반환할 토큰

  apiService = ApiService(
    baseUrl: 'https://api.example.com',
    tokenRefresher: fakeTokenRefresher,
    client: mockClient,
  );

  final response = await apiService.get('/v1/devices');

  expect(response.statusCode, equals(200));
  expect(callCount, equals(2)); // 두 번 호출됐는지 확인
  expect(fakeTokenRefresher.refreshCalled, isTrue);
});
```

처음엔 이 테스트에서 `callCount`가 계속 1로 나왔다. ApiService의 401 재시도 로직이 새 토큰을 `_getHeaders()`에서 다시 가져오는 게 아니라 기존 헤더를 그대로 재사용하고 있었다. 테스트가 없었으면 이 버그는 실제 토큰 만료 상황에서야 발견됐을 것이다. 한 시간 삽질하고 알아낸 거다.

## 네트워크 오류 케이스

`MockClient` 콜백에서 예외를 던지면 `SocketException` 시나리오를 재현할 수 있다.

```dart
import 'dart:io';

test('네트워크 오류 시 커스텀 예외로 변환', () async {
  final mockClient = MockClient((request) async {
    throw const SocketException('네트워크 연결 없음');
  });

  apiService = ApiService(
    baseUrl: 'https://api.example.com',
    tokenRefresher: fakeTokenRefresher,
    client: mockClient,
  );

  expect(
    () => apiService.get('/v1/devices'),
    throwsA(isA<NetworkException>()), // ApiService에서 감싼 커스텀 예외
  );
});
```

IoT 앱에서 이게 중요한 이유가 있다. 보일러 제어 명령이 네트워크 오류로 실패했는데 사용자에게 아무 피드백이 없으면 최악이다. `SocketException`을 `NetworkException`으로 변환해서 UI 레이어까지 전달하는 체인이 끊기지 않는지 이 테스트로 확인한다.

## FakeTokenRefresher

```dart
class FakeTokenRefresher implements TokenRefresher {
  String nextToken = 'fake-access-token';
  bool refreshCalled = false;

  @override
  Future<String> getValidAccessToken() async => nextToken;

  @override
  Future<void> refresh() async {
    refreshCalled = true;
  }
}
```

MockClient와 FakeTokenRefresher 조합으로 실제 서버, 실제 네트워크, 실제 키체인 없이 ApiService 동작 전체를 테스트할 수 있다.

## 여러 요청 순서 검증

IoT 앱에서는 기기 목록 조회 → 상태 조회 순으로 연속 요청이 발생한다. 이 순서를 검증하려면 MockClient를 상태 기반으로 작성하면 된다.

```dart
test('기기 목록 조회 후 상태 조회 순서 보장', () async {
  final requestPaths = <String>[];

  final mockClient = MockClient((request) async {
    requestPaths.add(request.url.path);
    if (request.url.path == '/v1/devices') {
      return http.Response('[{"id": "d1"}]', 200,
          headers: {'content-type': 'application/json'});
    }
    if (request.url.path.startsWith('/v1/devices/')) {
      return http.Response('{"status": "on"}', 200,
          headers: {'content-type': 'application/json'});
    }
    return http.Response('', 404);
  });

  // ... 테스트 실행 후

  expect(requestPaths, equals(['/v1/devices', '/v1/devices/d1/status']));
});
```

---

이걸 짜고 나서 ApiService 테스트 커버리지가 0%에서 87%로 올라갔다. 기존엔 BLE나 MQTT 격리 테스트만 신경 쓰고 HTTP 레이어는 테스트가 아예 없었다. 가장 자주 바뀌는 레이어인데 가장 테스트가 없었다. MockClient 주입 한 줄이 이걸 뒤집었다.

다음 편은 `flutter_secure_storage` 테스트 격리를 다룬다. JWT 토큰을 OS 키체인에 저장하는 플러그인이라 [Platform Channel mock]({% post_url 2026-06-27-10-00-00-827115-flutter-platform-channel-mock-test-methodchannel %})에서 다뤘던 `TestDefaultBinaryMessengerBinding` 패턴을 그대로 적용한다.

**참고 링크**
- [http 패키지 MockClient API](https://pub.dev/documentation/http/latest/testing/MockClient-class.html)
- [http REST API 구현 포스트]({% post_url 2026-05-21-10-34-46-271590-http-rest-api-communication-without-dio %})
- [JWT 자동갱신 포스트]({% post_url 2026-05-20-09-27-33-259245-jwt-token-auto-refresh-session-management %})
