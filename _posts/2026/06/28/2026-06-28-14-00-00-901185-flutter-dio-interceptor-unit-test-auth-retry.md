---
layout: post
title: "Dio 인터셉터 단위 테스트 — 401 자동갱신과 재시도 로직을 Flutter 없이 검증하는 법"
description: "Dio의 인증 인터셉터(401 자동갱신 + 재시도)를 단위 테스트하는 방법을 실제 IoT 앱 코드로 정리했다. http_mock_adapter로 Dio를 격리하고, 토큰 갱신 성공/실패 시나리오를 네트워크 없이 검증한다."
date: 2026-06-28
tags: [Flutter, Dart, CleanArchitecture, mqtt5_client, IoT, JWT, SecureStorage]
comments: true
share: true
---

![Dio HTTP 인터셉터 단위 테스트](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

IoT 앱에서 API 요청이 실패할 때 가장 흔한 시나리오는 401이다. 액세스 토큰이 만료됐을 때 인터셉터가 자동으로 리프레시하고, 원래 요청을 재시도하는 로직 — 이게 생각보다 테스트하기 까다롭다.

처음엔 그냥 통합 테스트로 커버하려 했는데, 토큰 갱신 API가 실패하는 케이스를 재현하려면 실제 서버 상태를 건드려야 해서 불가능했다. `http_mock_adapter`를 써서 Dio를 완전히 격리한 뒤에야 제대로 된 테스트를 쓸 수 있었다.

## 왜 Dio 인터셉터 테스트가 별도로 필요한가

이전 포스트에서 `MockClient`로 `http` 패키지를 격리하는 방법을 다뤘다. 그런데 Dio는 다르다. Dio의 인터셉터는 `onRequest`, `onResponse`, `onError`를 체인으로 처리하는 구조라, 단순히 클라이언트를 바꾸는 것만으로는 인터셉터 자체를 테스트할 수 없다.

특히 401 재시도 인터셉터는 내부에서 새로운 Dio 요청(토큰 갱신)을 보내고, 그 결과에 따라 원래 요청을 다시 실행하는 복잡한 흐름이다. 이걸 재현하려면 Dio 레벨에서 응답을 조작할 수 있어야 한다.

## 프로젝트 구조

IoT 앱의 API 레이어는 이런 구조다.

```
lib/
├── core/
│   └── network/
│       ├── dio_client.dart          # Dio 인스턴스 생성, 인터셉터 등록
│       ├── auth_interceptor.dart    # 401 감지 → 토큰 갱신 → 재시도
│       └── logging_interceptor.dart
├── data/
│   └── datasources/
│       └── device_remote_datasource.dart
test/
└── core/
    └── network/
        └── auth_interceptor_test.dart
```

## AuthInterceptor 구현

실제 운영 중인 인터셉터다. 401이 오면 `TokenRepository`로 갱신을 시도하고, 성공하면 원래 요청 헤더를 교체해서 재시도한다.

```dart
class AuthInterceptor extends Interceptor {
  final TokenRepository tokenRepository;
  final Dio dio;

  AuthInterceptor({required this.tokenRepository, required this.dio});

  @override
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await tokenRepository.getAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode != 401) {
      handler.next(err);
      return;
    }

    try {
      final newToken = await tokenRepository.refreshToken();
      final opts = err.requestOptions;
      opts.headers['Authorization'] = 'Bearer $newToken';

      // 원래 요청 재시도
      final response = await dio.fetch(opts);
      handler.resolve(response);
    } catch (e) {
      // 갱신 실패 → 로그아웃 처리
      await tokenRepository.clearTokens();
      handler.next(err);
    }
  }
}
```

## http_mock_adapter 설정

`http_mock_adapter` 패키지를 쓰면 Dio 내부 HttpAdapter를 교체해서 실제 요청 없이 응답을 제어할 수 있다.

```yaml
# pubspec.yaml
dev_dependencies:
  http_mock_adapter: ^0.6.1
  mocktail: ^1.0.4
```

테스트 파일 기본 구조:

```dart
import 'package:dio/dio.dart';
import 'package:http_mock_adapter/http_mock_adapter.dart';
import 'package:mocktail/mocktail.dart';
import 'package:test/test.dart';

class MockTokenRepository extends Mock implements TokenRepository {}

void main() {
  late Dio dio;
  late DioAdapter dioAdapter;
  late MockTokenRepository mockTokenRepo;
  late AuthInterceptor authInterceptor;

  setUp(() {
    dio = Dio(BaseOptions(baseUrl: 'https://api.example.com'));
    dioAdapter = DioAdapter(dio: dio, matcher: const FullHttpRequestMatcher());
    mockTokenRepo = MockTokenRepository();
    authInterceptor = AuthInterceptor(tokenRepository: mockTokenRepo, dio: dio);
    dio.interceptors.add(authInterceptor);
  });

  tearDown(() {
    dioAdapter.close();
  });
}
```

`FullHttpRequestMatcher`를 쓰지 않으면 path + method만 매칭해서 헤더가 다른 재시도 요청을 별도로 stub하기 어렵다. 처음에 이걸 몰라서 재시도 시나리오 테스트가 계속 터졌다.

## 시나리오 1 — 정상 요청 (토큰 주입 확인)

요청마다 Authorization 헤더가 붙는지 확인한다.

```dart
test('정상 요청에 Authorization 헤더가 주입된다', () async {
  when(() => mockTokenRepo.getAccessToken())
      .thenAnswer((_) async => 'valid-access-token');

  dioAdapter.onGet(
    '/devices',
    (server) => server.reply(200, {'devices': []}),
  );

  final response = await dio.get('/devices');

  expect(response.statusCode, 200);
  expect(
    response.requestOptions.headers['Authorization'],
    'Bearer valid-access-token',
  );
});
```

## 시나리오 2 — 401 → 토큰 갱신 성공 → 재시도 성공

핵심 시나리오다. 같은 엔드포인트에서 첫 응답은 401, 두 번째 응답은 200을 돌려줘야 한다.

```dart
test('401 응답 후 토큰 갱신 성공 시 원래 요청을 재시도한다', () async {
  when(() => mockTokenRepo.getAccessToken())
      .thenAnswer((_) async => 'expired-token');
  when(() => mockTokenRepo.refreshToken())
      .thenAnswer((_) async => 'new-access-token');

  // 첫 번째 요청: 401
  dioAdapter.onGet(
    '/devices',
    (server) => server.reply(401, {'message': 'Unauthorized'}),
    headers: {'Authorization': 'Bearer expired-token'},
  );

  // 재시도 요청 (새 토큰): 200
  dioAdapter.onGet(
    '/devices',
    (server) => server.reply(200, {'devices': ['boiler-01']}),
    headers: {'Authorization': 'Bearer new-access-token'},
  );

  final response = await dio.get('/devices');

  expect(response.statusCode, 200);
  expect(response.data['devices'], contains('boiler-01'));
  verify(() => mockTokenRepo.refreshToken()).called(1);
});
```

## 시나리오 3 — 401 → 토큰 갱신 실패 → 로그아웃

갱신 API 자체가 실패하는 케이스다. `clearTokens()`가 호출됐는지, 예외가 전파됐는지를 확인한다.

```dart
test('토큰 갱신 실패 시 토큰을 지우고 예외를 전파한다', () async {
  when(() => mockTokenRepo.getAccessToken())
      .thenAnswer((_) async => 'expired-token');
  when(() => mockTokenRepo.refreshToken())
      .thenThrow(Exception('Refresh token expired'));
  when(() => mockTokenRepo.clearTokens()).thenAnswer((_) async {});

  dioAdapter.onGet(
    '/devices',
    (server) => server.reply(401, {'message': 'Unauthorized'}),
  );

  await expectLater(
    () => dio.get('/devices'),
    throwsA(isA<DioException>()),
  );

  verify(() => mockTokenRepo.clearTokens()).called(1);
});
```

## 삽질했던 부분

**dioAdapter는 등록 순서대로 소비된다.** 같은 경로에 다른 응답을 순서대로 등록하면 첫 번째 요청이 첫 번째 stub을, 두 번째 요청이 두 번째 stub을 소비한다. 근데 헤더를 매칭하지 않으면 재시도 요청도 같은 stub을 쓰게 돼서 무한루프처럼 동작한다.

`FullHttpRequestMatcher`로 헤더까지 매칭하거나, 응답을 queue 방식으로 등록하는 방법 중 하나를 택해야 한다.

**dio.fetch()로 재시도할 때 인터셉터가 다시 타지 않도록 주의.** `dio.fetch(opts)`는 인터셉터를 건너뛰지 않아서, 재시도가 또 401을 받으면 무한 재귀가 발생할 수 있다. 실제로 한 번 겪었다. 재시도 요청에는 플래그를 달아두는 게 안전하다.

```dart
// onError 안에서
if (err.requestOptions.extra['retried'] == true) {
  handler.next(err); // 이미 재시도한 요청 → 그냥 에러 전파
  return;
}
err.requestOptions.extra['retried'] = true;
```

---

다음 편은 `easy_localization` 다국어 처리를 단위 테스트에서 격리하는 방법이다. 로케일을 런타임에 바꾸는 로직이 테스트에서 어떻게 동작하는지 다룬다.

**참고 링크**
- [http_mock_adapter 패키지](https://pub.dev/packages/http_mock_adapter)
- [Dio 공식 인터셉터 문서](https://pub.dev/packages/dio#interceptors)
- [mocktail 패키지](https://pub.dev/packages/mocktail)
