---
layout: post
title: "flutter_secure_storage 단위 테스트 격리 — FakeSecureStorage와 JWT 토큰 테스트"
description: "flutter_secure_storage를 사용하는 코드는 단위 테스트에서 MissingPluginException이 터진다. 추상화 레이어를 끼워 FakeSecureStorageService로 격리하고, JWT TokenRepository를 단위 테스트하는 방법을 실전 코드로 정리했다."
date: 2026-06-27
tags: [Flutter, flutter_secure_storage, JWT, CleanArchitecture, SecureStorage]
comments: true
share: true
---

![Flutter 보안 스토리지 단위 테스트](https://images.unsplash.com/photo-1555949963-ff9fe0c870eb?w=800&q=80)

`flutter_secure_storage`로 JWT 토큰을 저장하는 코드를 짜고, 단위 테스트를 붙이려는 순간 이게 뜬다.

```
MissingPluginException(No implementation found for method
containsKey on channel plugins.it_nomads.com/flutter_secure_storage)
```

네이티브 채널(iOS Keychain, Android Keystore)을 타기 때문에 Flutter 런타임 없이는 동작하지 않는다. 이걸 단위 테스트에서 돌리는 방법은 하나다 — 추상화 레이어를 끼우고 테스트에선 메모리 구현체로 바꿔치기하는 것.

## 문제의 구조

이 블로그의 스마트홈 앱에서 JWT 관련 코드는 이렇게 생겼다.

```dart
class TokenRepository {
  final FlutterSecureStorage _storage = FlutterSecureStorage();

  Future<String?> getAccessToken() async {
    return await _storage.read(key: 'access_token');
  }

  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await _storage.write(key: 'access_token', value: accessToken);
    await _storage.write(key: 'refresh_token', value: refreshToken);
  }

  Future<void> clearTokens() async {
    await _storage.deleteAll();
  }
}
```

`FlutterSecureStorage()`를 내부에서 직접 생성하면 테스트에서 교체할 방법이 없다. 여기서 MissingPluginException이 터진다.

## 추상화 레이어 끼우기

`FlutterSecureStorage` 자체는 인터페이스를 노출하지 않는다. 그래서 직접 만들어야 한다.

```dart
// lib/core/storage/secure_storage_service.dart
abstract class SecureStorageService {
  Future<String?> read(String key);
  Future<void> write(String key, String value);
  Future<void> delete(String key);
  Future<void> deleteAll();
  Future<bool> containsKey(String key);
}
```

실제 구현체는 `flutter_secure_storage`를 감싼다.

```dart
// lib/core/storage/flutter_secure_storage_service.dart
class FlutterSecureStorageService implements SecureStorageService {
  final FlutterSecureStorage _storage;

  FlutterSecureStorageService({
    FlutterSecureStorage? storage,
  }) : _storage = storage ?? const FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );

  @override
  Future<String?> read(String key) => _storage.read(key: key);

  @override
  Future<void> write(String key, String value) =>
      _storage.write(key: key, value: value);

  @override
  Future<void> delete(String key) => _storage.delete(key: key);

  @override
  Future<void> deleteAll() => _storage.deleteAll();

  @override
  Future<bool> containsKey(String key) => _storage.containsKey(key: key);
}
```

## FakeSecureStorageService — 메모리 구현체

테스트용은 단순하다. `Map<String, String>`에 넣고 꺼내면 된다.

```dart
// test/fakes/fake_secure_storage_service.dart
class FakeSecureStorageService implements SecureStorageService {
  final Map<String, String> _store = {};

  @override
  Future<String?> read(String key) async => _store[key];

  @override
  Future<void> write(String key, String value) async {
    _store[key] = value;
  }

  @override
  Future<void> delete(String key) async {
    _store.remove(key);
  }

  @override
  Future<void> deleteAll() async {
    _store.clear();
  }

  @override
  Future<bool> containsKey(String key) async => _store.containsKey(key);

  // 테스트에서 내부 상태 확인용
  Map<String, String> get dump => Map.unmodifiable(_store);
}
```

`dump` getter는 "write가 실제로 호출됐는지"를 검증할 때 쓴다. Spy처럼 활용할 수 있다.

## TokenRepository 수정

이제 `TokenRepository`가 `SecureStorageService`를 주입받도록 바꾼다.

```dart
// lib/features/auth/data/repositories/token_repository.dart
class TokenRepository {
  static const _accessKey = 'access_token';
  static const _refreshKey = 'refresh_token';

  final SecureStorageService _storage;

  TokenRepository({required SecureStorageService storage})
      : _storage = storage;

  Future<String?> getAccessToken() => _storage.read(_accessKey);
  Future<String?> getRefreshToken() => _storage.read(_refreshKey);

  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await _storage.write(_accessKey, accessToken);
    await _storage.write(_refreshKey, refreshToken);
  }

  Future<void> clearTokens() => _storage.deleteAll();

  Future<bool> hasValidToken() => _storage.containsKey(_accessKey);
}
```

GetX에서 등록할 때는 실제 구현체를 넣는다.

```dart
// lib/core/bindings/app_binding.dart
Get.put<SecureStorageService>(FlutterSecureStorageService());
Get.put(TokenRepository(storage: Get.find<SecureStorageService>()));
```

## 단위 테스트 작성

이제 `flutter_secure_storage`가 전혀 없어도 테스트가 돌아간다.

```dart
// test/features/auth/token_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:my_app/features/auth/data/repositories/token_repository.dart';
import '../../fakes/fake_secure_storage_service.dart';

void main() {
  late FakeSecureStorageService fakeStorage;
  late TokenRepository sut;

  setUp(() {
    fakeStorage = FakeSecureStorageService();
    sut = TokenRepository(storage: fakeStorage);
  });

  group('TokenRepository', () {
    test('저장 후 accessToken을 읽을 수 있다', () async {
      await sut.saveTokens(
        accessToken: 'eyJhbGciOiJIUzI1NiJ9.test',
        refreshToken: 'refresh_abc123',
      );

      final token = await sut.getAccessToken();

      expect(token, equals('eyJhbGciOiJIUzI1NiJ9.test'));
    });

    test('saveTokens는 두 키 모두 스토리지에 쓴다', () async {
      await sut.saveTokens(
        accessToken: 'access_abc',
        refreshToken: 'refresh_xyz',
      );

      expect(fakeStorage.dump['access_token'], equals('access_abc'));
      expect(fakeStorage.dump['refresh_token'], equals('refresh_xyz'));
    });

    test('clearTokens 후 hasValidToken이 false를 반환한다', () async {
      await sut.saveTokens(
        accessToken: 'access_abc',
        refreshToken: 'refresh_xyz',
      );
      await sut.clearTokens();

      final hasToken = await sut.hasValidToken();

      expect(hasToken, isFalse);
    });

    test('토큰 없으면 getAccessToken은 null 반환', () async {
      final token = await sut.getAccessToken();
      expect(token, isNull);
    });

    test('clearTokens는 refreshToken도 지운다', () async {
      await sut.saveTokens(
        accessToken: 'access_abc',
        refreshToken: 'refresh_xyz',
      );
      await sut.clearTokens();

      final refreshToken = await sut.getRefreshToken();
      expect(refreshToken, isNull);
    });
  });
}
```

![FakeSecureStorage 단위 테스트 패턴](https://images.unsplash.com/photo-1516116216624-53e697fedbea?w=800&q=80)

## JWT 자동갱신 서비스 테스트

`TokenRepository`를 감싸는 `TokenService`가 있을 경우 — 만료 판단, 갱신 요청까지 포함해서 테스트할 수 있다.

```dart
// lib/features/auth/domain/services/token_service.dart
class TokenService {
  final TokenRepository _tokenRepo;
  final AuthApiService _authApi;

  TokenService({
    required TokenRepository tokenRepo,
    required AuthApiService authApi,
  })  : _tokenRepo = tokenRepo,
        _authApi = authApi;

  Future<String?> getValidAccessToken() async {
    final token = await _tokenRepo.getAccessToken();
    if (token == null) return null;
    if (_isExpired(token)) {
      return await _refresh();
    }
    return token;
  }

  Future<String?> _refresh() async {
    final refreshToken = await _tokenRepo.getRefreshToken();
    if (refreshToken == null) return null;

    final newTokens = await _authApi.refreshToken(refreshToken);
    if (newTokens == null) {
      await _tokenRepo.clearTokens();
      return null;
    }

    await _tokenRepo.saveTokens(
      accessToken: newTokens.accessToken,
      refreshToken: newTokens.refreshToken,
    );
    return newTokens.accessToken;
  }

  bool _isExpired(String token) {
    // JWT payload 파싱해서 exp 확인 (생략)
    return false;
  }
}
```

`AuthApiService`도 Fake로 만들면 완전한 격리 테스트가 된다.

```dart
// test/fakes/fake_auth_api_service.dart
class FakeAuthApiService implements AuthApiService {
  TokenPair? refreshResult;
  int refreshCallCount = 0;

  @override
  Future<TokenPair?> refreshToken(String refreshToken) async {
    refreshCallCount++;
    return refreshResult;
  }
}
```

```dart
test('refreshToken 호출 성공 시 새 토큰이 저장된다', () async {
  final fakeStorage = FakeSecureStorageService();
  final fakeApi = FakeAuthApiService()
    ..refreshResult = TokenPair(
      accessToken: 'new_access',
      refreshToken: 'new_refresh',
    );
  final tokenRepo = TokenRepository(storage: fakeStorage);
  final sut = TokenService(tokenRepo: tokenRepo, authApi: fakeApi);

  // 만료된 토큰이 저장된 상태 (실제로는 exp가 지난 JWT)
  await tokenRepo.saveTokens(
    accessToken: 'expired_token',
    refreshToken: 'valid_refresh',
  );

  // _isExpired를 항상 true로 만들려면 TokenService를 조금 수정하거나
  // expiredAt을 주입받는 방식으로 설계할 수 있다

  expect(fakeApi.refreshCallCount, 0); // 아직 갱신 안 했음
});
```

## 삽질했던 부분

`FlutterSecureStorage`의 `AndroidOptions`나 `IOSOptions`를 쓰면서 `FlutterSecureStorageService` 생성자에서 옵션을 잘못 전달해 테스트에서 실제 `FlutterSecureStorage`가 살아있는 경우가 있었다. 의존성 주입이 제대로 안 된 것.

GetX에서 `Get.find<SecureStorageService>()`로 잘 쓰이고 있는지 확인하려면:

```dart
// 테스트에서 GetX 바인딩을 통째로 교체하는 방법
setUp(() {
  Get.put<SecureStorageService>(FakeSecureStorageService());
  Get.put(TokenRepository(storage: Get.find<SecureStorageService>()));
});

tearDown(() {
  Get.reset();
});
```

이렇게 하면 Widget Test나 Controller Test에서도 동일한 패턴으로 재사용할 수 있다.

또 한 가지 — `deleteAll()`이 `delete(key)` 여러 번 호출한 것과 같은 결과를 내는지 명시적으로 테스트해두는 게 좋다. 실제 Keystore 구현에선 `deleteAll`이 다른 앱의 Shared Preferences까지 지우는 버그가 있었다. Fake에선 `_store.clear()`라 범위가 명확하지만, 실제 환경 차이를 인지하고 있어야 한다.

---

다음 편은 GetX Bindings에서 의존성 주입을 테스트 환경에 맞게 교체하는 전략을 다룬다. `Get.put`/`Get.lazyPut`/`Get.replace`를 테스트에서 어떻게 제어할 것인지.

**참고 링크**
- [flutter_secure_storage pub.dev](https://pub.dev/packages/flutter_secure_storage)
- [flutter_secure_storage GitHub — 테스트 이슈 논의](https://github.com/mogol/flutter_secure_storage/issues/546)
- [JWT 자동갱신 실전]({% post_url 2026-05-20-09-27-33-259245-jwt-token-auto-refresh-session-management %})
- [flutter_secure_storage 민감 데이터 저장]({% post_url 2026-06-05-10-19-01-456765-flutter-secure-storage-sensitive-data %})
- [테스트 더블 패턴 비교]({% post_url 2026-06-26-12-00-00-765390-flutter-test-doubles-fake-mock-stub-spy-patterns %})
