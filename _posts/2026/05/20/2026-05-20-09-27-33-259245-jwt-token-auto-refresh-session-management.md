---
layout: post
title: "JWT 토큰 자동 갱신 - 끊기지 않는 세션 관리"
description: " "
date: 2026-05-20
tags: [Flutter, JWT, 토큰관리, 인증, 세션]
comments: true
share: true
---
![인증 보안](https://images.unsplash.com/photo-1614064641938-3bbee52942c7?w=800&q=80)

## JWT 기본 구조

Access Token과 Refresh Token 두 가지를 사용한다:

- **Access Token**: API 호출 시 인증에 사용. 유효 기간 짧음 (예: 1시간)
- **Refresh Token**: Access Token 갱신에 사용. 유효 기간 김 (예: 30일)

Access Token이 만료됐을 때 Refresh Token으로 새 Access Token을 받는 구조다.

## TokenManager 구현

```dart
// data/auth/token_manager.dart
class TokenManager {
  static const _accessTokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';
  static const _expiresAtKey = 'token_expires_at';

  final FlutterSecureStorage _storage;

  TokenManager(this._storage);

  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
    required DateTime expiresAt,
  }) async {
    await Future.wait([
      _storage.write(key: _accessTokenKey, value: accessToken),
      _storage.write(key: _refreshTokenKey, value: refreshToken),
      _storage.write(
        key: _expiresAtKey,
        value: expiresAt.millisecondsSinceEpoch.toString(),
      ),
    ]);
  }

  Future<String?> getAccessToken() => _storage.read(key: _accessTokenKey);
  Future<String?> getRefreshToken() => _storage.read(key: _refreshTokenKey);

  Future<bool> isAccessTokenExpired() async {
    final expiresAtStr = await _storage.read(key: _expiresAtKey);
    if (expiresAtStr == null) return true;
    
    final expiresAt = DateTime.fromMillisecondsSinceEpoch(
      int.parse(expiresAtStr),
    );
    // 만료 5분 전부터 갱신 시도
    return DateTime.now().isAfter(expiresAt.subtract(const Duration(minutes: 5)));
  }

  Future<void> clearTokens() async {
    await Future.wait([
      _storage.delete(key: _accessTokenKey),
      _storage.delete(key: _refreshTokenKey),
      _storage.delete(key: _expiresAtKey),
    ]);
  }
}
```

## TokenRefresher - 갱신 로직

```dart
// data/auth/token_refresher.dart
class TokenRefresher {
  final TokenManager _tokenManager;
  final AuthRefreshDataSource _dataSource;

  // 동시에 여러 요청이 갱신을 시도할 때 중복 방지용
  Completer<String>? _refreshCompleter;

  TokenRefresher(this._tokenManager, this._dataSource);

  Future<String> getValidAccessToken() async {
    // 유효한 토큰이 있으면 바로 반환
    if (!await _tokenManager.isAccessTokenExpired()) {
      return (await _tokenManager.getAccessToken())!;
    }

    // 이미 갱신 중이면 완료 대기
    if (_refreshCompleter != null && !_refreshCompleter!.isCompleted) {
      return _refreshCompleter!.future;
    }

    // 새 갱신 시작
    _refreshCompleter = Completer<String>();
    
    try {
      final refreshToken = await _tokenManager.getRefreshToken();
      if (refreshToken == null) throw UnauthorizedException();

      final response = await _dataSource.refreshToken(refreshToken);
      
      await _tokenManager.saveTokens(
        accessToken: response.accessToken,
        refreshToken: response.refreshToken,
        expiresAt: response.expiresAt,
      );
      
      _refreshCompleter!.complete(response.accessToken);
      return response.accessToken;
    } catch (e) {
      _refreshCompleter!.completeError(e);
      // 갱신 실패 시 로그아웃
      await _handleRefreshFailure();
      rethrow;
    } finally {
      _refreshCompleter = null;
    }
  }

  Future<void> _handleRefreshFailure() async {
    await _tokenManager.clearTokens();
    // 로그인 화면으로 이동
    Get.offAllNamed(Routes.signin);
  }
}
```

## API 서비스에서 자동 적용

모든 API 요청에 토큰을 자동으로 붙이고, 401이 오면 자동으로 갱신한다:

```dart
// data/services/api_service.dart
class ApiService {
  final TokenRefresher _tokenRefresher;

  Future<Response> get(String path) async {
    final token = await _tokenRefresher.getValidAccessToken();
    
    try {
      final response = await http.get(
        Uri.parse('$baseUrl$path'),
        headers: {
          'Authorization': 'Bearer $token',
          'Content-Type': 'application/json',
        },
      );
      
      if (response.statusCode == 401) {
        // 토큰 갱신 후 재시도
        final newToken = await _tokenRefresher.getValidAccessToken();
        return http.get(
          Uri.parse('$baseUrl$path'),
          headers: {'Authorization': 'Bearer $newToken'},
        );
      }
      
      return response;
    } catch (e) {
      rethrow;
    }
  }
}
```

## 앱 시작 시 토큰 상태 확인

앱이 시작될 때 저장된 토큰이 유효한지 확인해서 로그인/메인 화면을 결정한다:

```dart
// core/services/auth_manager.dart
class AuthManager extends GetxController {
  final TokenManager _tokenManager;
  
  Future<void> checkAuthState() async {
    final accessToken = await _tokenManager.getAccessToken();
    final refreshToken = await _tokenManager.getRefreshToken();
    
    if (accessToken == null || refreshToken == null) {
      Get.offAllNamed(Routes.signin);
      return;
    }
    
    // Refresh Token 유효성 확인
    try {
      await _tokenRefresher.getValidAccessToken();
      Get.offAllNamed(Routes.mainTab);
    } catch (_) {
      Get.offAllNamed(Routes.signin);
    }
  }
}
```

---

다음 편은 Dio 없이 http 패키지로 REST API를 구현하는 방법이다.
