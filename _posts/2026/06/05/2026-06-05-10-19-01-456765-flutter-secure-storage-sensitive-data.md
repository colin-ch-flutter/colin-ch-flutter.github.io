---
layout: post
title: "flutter_secure_storage로 민감 정보 안전하게 저장"
description: " "
date: 2026-06-05
tags: [Flutter, SecureStorage, 보안, 토큰, Keychain]
comments: true
share: true
---

# flutter_secure_storage로 민감 정보 안전하게 저장

![보안 저장소](https://images.unsplash.com/photo-1614064641938-3bbee52942c7?w=800&q=80)

## SharedPreferences는 왜 안 되나

`SharedPreferences`에 저장된 데이터는 암호화되지 않는다. Android에서는 `/data/data/{package}/shared_prefs/`에 평문으로 저장되기 때문에, 루팅된 기기나 백업 파일에서 데이터가 노출될 수 있다.

JWT Access Token, Refresh Token 같은 민감한 인증 정보는 반드시 암호화된 저장소에 보관해야 한다.

## flutter_secure_storage

`flutter_secure_storage`는 플랫폼별 보안 저장소를 사용한다:

- **iOS**: Keychain Services
- **Android**: Android Keystore + EncryptedSharedPreferences

```yaml
dependencies:
  flutter_secure_storage: ^9.2.4
```

## 기본 사용법

```dart
final storage = const FlutterSecureStorage();

// 저장
await storage.write(key: 'access_token', value: token);

// 읽기
final token = await storage.read(key: 'access_token');

// 삭제
await storage.delete(key: 'access_token');

// 모두 삭제 (로그아웃 시)
await storage.deleteAll();
```

## TokenManager 구현

이전 편에서 소개한 TokenManager를 flutter_secure_storage 기반으로 구현한다:

```dart
// data/auth/token_manager.dart
class TokenManager {
  final FlutterSecureStorage _storage;

  static const _keyAccessToken = 'smarthome_access_token';
  static const _keyRefreshToken = 'smarthome_refresh_token';
  static const _keyExpiresAt = 'smarthome_token_expires_at';
  static const _keyUserId = 'smarthome_user_id';

  const TokenManager(this._storage);

  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
    required DateTime expiresAt,
    required String userId,
  }) async {
    await Future.wait([
      _storage.write(key: _keyAccessToken, value: accessToken),
      _storage.write(key: _keyRefreshToken, value: refreshToken),
      _storage.write(
        key: _keyExpiresAt,
        value: expiresAt.millisecondsSinceEpoch.toString(),
      ),
      _storage.write(key: _keyUserId, value: userId),
    ]);
  }

  Future<String?> getAccessToken() async {
    return _storage.read(key: _keyAccessToken);
  }

  Future<String?> getRefreshToken() async {
    return _storage.read(key: _keyRefreshToken);
  }

  Future<String?> getUserId() async {
    return _storage.read(key: _keyUserId);
  }

  Future<bool> hasValidTokens() async {
    final accessToken = await getAccessToken();
    final refreshToken = await getRefreshToken();
    return accessToken != null && refreshToken != null;
  }

  Future<void> clearAll() async {
    await _storage.deleteAll();
  }
}
```

## Android 백업 설정

Android 자동 백업 기능이 켜져 있으면 Secure Storage 데이터가 다른 기기로 복원될 수 있다. 보안을 위해 백업에서 제외한다:

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<application
    android:allowBackup="true"
    android:fullBackupContent="@xml/backup_rules">
```

```xml
<!-- android/app/src/main/res/xml/backup_rules.xml -->
<?xml version="1.0" encoding="utf-8"?>
<full-backup-content>
    <exclude domain="sharedpref" path="FlutterSecureStorage"/>
</full-backup-content>
```

## 앱 삭제 후 재설치 동작

iOS에서는 앱을 삭제해도 Keychain 데이터가 남아있다. 재설치 후 이전 토큰이 그대로 남아있어서 예상치 못한 동작이 생길 수 있다.

처음 앱을 실행할 때 이전 설치의 데이터인지 확인하고 지우는 처리를 추가했다:

```dart
// 앱 첫 실행 여부 확인
Future<void> handleFirstLaunchCleanup() async {
  final isFirstLaunch = !GetStorage().hasData('app_launched');
  
  if (isFirstLaunch) {
    // 이전 설치 데이터 정리 (iOS Keychain은 앱 삭제 후에도 남음)
    await _tokenManager.clearAll();
    GetStorage().write('app_launched', true);
  }
}
```

## 보안 고려사항

Secure Storage를 쓴다고 해서 완벽한 보안이 되는 건 아니다. 루팅된 Android 기기나 탈옥된 iOS에서는 Keychain 데이터도 접근 가능할 수 있다. 그래도 일반 SharedPreferences보다는 훨씬 안전하다.

추가로 고려할 수 있는 보안 조치:
- 루팅/탈옥 기기 감지 후 경고
- 생체 인증으로 앱 잠금
- 토큰 유효 기간 짧게 유지

---

다음 편부터는 IoT 모니터링 대시보드 파트다. 산업용 IoT 기기를 앱에서 제어하는 이야기다.
