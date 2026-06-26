---
layout: post
title: "Flutter Firebase 단위 테스트 — firebase_core 없이 Auth·RemoteConfig Fake로 격리하기"
description: "Firebase.initializeApp() 없이도 단위 테스트를 돌리는 법. firebase_auth_mocks와 직접 만든 FakeRemoteConfig로 Firebase 의존성을 깔끔하게 격리하는 실전 패턴을 정리했다."
date: 2026-06-26
tags: [Flutter, Firebase, RemoteConfig, 인증, CleanArchitecture]
comments: true
share: true
---

# Flutter Firebase 단위 테스트 — firebase_core 없이 Auth·RemoteConfig Fake로 격리하기

![Firebase 테스트 격리 개념 — 클라우드 없이 로컬에서 테스트](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

`Firebase.initializeApp()`이 있는 클래스를 단위 테스트하려고 하면 처음엔 이런 에러가 뜬다.

```
No Firebase App '[DEFAULT]' has been created - call Firebase.initializeApp()
```

그래서 `setUp()`에 `Firebase.initializeApp()`을 넣으면 이번엔 플랫폼 채널이 없다고 터진다. 실기기나 에뮬레이터 없이는 Firebase 초기화가 안 되기 때문이다. 결국 "Firebase 쓰면 단위 테스트 못 한다"는 결론에 도달하는데, 그건 아키텍처 문제다. 격리만 제대로 하면 CI 리눅스 머신에서도 Firebase 관련 로직을 100% 단위 테스트할 수 있다.

## 왜 Firebase가 테스트를 방해하는가

Flutter Firebase SDK는 내부적으로 네이티브 플랫폼 채널을 통해 실제 Firebase SDK(Android/iOS)와 통신한다. `firebase_auth`, `firebase_remote_config` 모두 마찬가지다. 단위 테스트 환경은 플랫폼 채널이 없기 때문에 Firebase 인스턴스를 직접 생성하는 순간 바로 죽는다.

해결 방법은 단순하다. **Firebase를 직접 참조하지 말고 인터페이스만 의존하게 만들어라.** 그러면 테스트에선 Fake 구현체를 꽂으면 된다.

```
실제 코드:  Controller → IAuthService → FirebaseAuthService (실제 Firebase 호출)
테스트 코드: Controller → IAuthService → FakeAuthService (메모리 구현)
```

## Auth 격리 — firebase_auth_mocks 활용

`firebase_auth`는 `firebase_auth_mocks` 패키지를 쓰면 빠르다. 직접 만들 필요가 없다.

```yaml
# pubspec.yaml (dev_dependencies)
dev_dependencies:
  firebase_auth_mocks: ^0.14.0
  google_sign_in_mocks: ^0.3.0
```

하지만 이걸 Controller에서 직접 쓰면 안 된다. 이 패키지도 어디까지나 테스트 전용이라, `FirebaseAuth` 구체 타입을 직접 주입하는 구조면 프로덕션 코드에 test 패키지가 새어 들어간다.

먼저 인터페이스부터 정의한다.

```dart
// lib/domain/services/i_auth_service.dart
abstract class IAuthService {
  Future<String?> signIn(String email, String password);
  Future<void> signOut();
  Stream<bool> get authStateChanges;
}
```

Firebase 실구현체는 이 인터페이스를 implements한다.

```dart
// lib/data/services/firebase_auth_service.dart
class FirebaseAuthService implements IAuthService {
  final FirebaseAuth _auth;
  FirebaseAuthService(this._auth);

  @override
  Future<String?> signIn(String email, String password) async {
    final result = await _auth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
    return result.user?.uid;
  }

  @override
  Future<void> signOut() => _auth.signOut();

  @override
  Stream<bool> get authStateChanges =>
      _auth.authStateChanges().map((user) => user != null);
}
```

테스트용 Fake는 이렇게 만든다. `firebase_auth_mocks`를 안 쓰고 그냥 메모리 구현으로 만드는 게 더 명확할 때도 많다.

```dart
// test/fakes/fake_auth_service.dart
class FakeAuthService implements IAuthService {
  String? _currentUid;
  bool _shouldFail = false;

  void setUser(String uid) => _currentUid = uid;
  void setShouldFail(bool value) => _shouldFail = value;

  @override
  Future<String?> signIn(String email, String password) async {
    if (_shouldFail) throw Exception('로그인 실패');
    _currentUid = 'test-uid-$email';
    return _currentUid;
  }

  @override
  Future<void> signOut() async => _currentUid = null;

  @override
  Stream<bool> get authStateChanges =>
      Stream.value(_currentUid != null);
}
```

테스트는 깔끔하다.

```dart
void main() {
  late FakeAuthService fakeAuth;
  late AuthController controller;

  setUp(() {
    fakeAuth = FakeAuthService();
    controller = AuthController(authService: fakeAuth);
  });

  test('로그인 성공 시 isLoggedIn이 true', () async {
    await controller.login('user@example.com', '1234');
    expect(controller.isLoggedIn, isTrue);
  });

  test('로그인 실패 시 errorMessage 세팅', () async {
    fakeAuth.setShouldFail(true);
    await controller.login('bad@example.com', 'wrong');
    expect(controller.errorMessage, isNotNull);
  });
}
```

Firebase 초기화 없음. 플랫폼 채널 없음. 리눅스 CI에서 그냥 돌아간다.

## RemoteConfig 격리 — 직접 Fake 만들기

`firebase_remote_config`용 공식 Fake 패키지는 없다. 직접 만들어야 한다.

IoT 앱에서 RemoteConfig를 쓰는 상황은 주로 이것들이다.
- 기기 제어 허용 온도 범위 서버에서 관리
- 긴급 점검 시 앱 기능 차단 플래그
- A/B 테스트용 UI 파라미터

인터페이스를 먼저 뽑아낸다.

```dart
// lib/domain/services/i_remote_config_service.dart
abstract class IRemoteConfigService {
  Future<void> initialize();
  String getString(String key);
  bool getBool(String key);
  double getDouble(String key);
}
```

실구현체.

```dart
// lib/data/services/firebase_remote_config_service.dart
class FirebaseRemoteConfigService implements IRemoteConfigService {
  final FirebaseRemoteConfig _config;

  FirebaseRemoteConfigService(this._config);

  @override
  Future<void> initialize() async {
    await _config.setConfigSettings(RemoteConfigSettings(
      fetchTimeout: const Duration(seconds: 10),
      minimumFetchInterval: const Duration(hours: 1),
    ));
    await _config.fetchAndActivate();
  }

  @override
  String getString(String key) => _config.getString(key);

  @override
  bool getBool(String key) => _config.getBool(key);

  @override
  double getDouble(String key) => _config.getDouble(key);
}
```

Fake 구현체. Map에다 기본값 세팅하고, 테스트마다 원하는 값 꽂아 넣는 구조다.

```dart
// test/fakes/fake_remote_config_service.dart
class FakeRemoteConfigService implements IRemoteConfigService {
  final Map<String, dynamic> _values = {
    'max_temp': 30.0,
    'min_temp': 10.0,
    'maintenance_mode': false,
    'welcome_message': '홈 IoT에 오신 것을 환영합니다',
  };

  void setValue(String key, dynamic value) => _values[key] = value;

  @override
  Future<void> initialize() async {} // no-op

  @override
  String getString(String key) => _values[key] as String? ?? '';

  @override
  bool getBool(String key) => _values[key] as bool? ?? false;

  @override
  double getDouble(String key) => (_values[key] as num?)?.toDouble() ?? 0.0;
}
```

이걸로 Controller 테스트를 짜면 이런 게 된다.

```dart
void main() {
  late FakeRemoteConfigService fakeConfig;
  late DeviceController controller;

  setUp(() {
    fakeConfig = FakeRemoteConfigService();
    controller = DeviceController(remoteConfig: fakeConfig);
  });

  test('점검 모드일 때 제어 시도 시 차단', () async {
    fakeConfig.setValue('maintenance_mode', true);

    final result = await controller.setTemperature(25.0);

    expect(result, equals(ControlResult.blocked));
    expect(controller.errorReason, contains('점검'));
  });

  test('온도 범위 초과 시 거부', () async {
    fakeConfig.setValue('max_temp', 28.0);

    final result = await controller.setTemperature(35.0);

    expect(result, equals(ControlResult.outOfRange));
  });
}
```

점검 모드 로직, 온도 범위 검증 로직 — Firebase 없이 전부 단위 테스트 가능하다.

## Crashlytics — 테스트에서 완전히 무시하기

![에러 로그 격리 — 테스트 중엔 Crashlytics가 개입하면 안 된다](https://images.unsplash.com/photo-1555949963-aa79dcee981c?w=800&q=80)

Crashlytics는 좀 다르다. 에러를 기록하는 단방향 서비스라 테스트에서 검증할 내용이 사실 별로 없다. `recordError()`가 몇 번 호출됐는지 확인하는 게 의미 있는 경우는 드물다.

그냥 no-op Fake로 만들어둔다.

```dart
// test/fakes/fake_crashlytics_service.dart
class FakeCrashlyticsService implements ICrashlyticsService {
  final List<dynamic> recordedErrors = [];

  @override
  Future<void> recordError(dynamic exception, StackTrace? stack) async {
    recordedErrors.add(exception); // 필요하면 검증용으로 쌓아둠
  }

  @override
  Future<void> setUserId(String userId) async {} // no-op

  @override
  Future<void> log(String message) async {} // no-op
}
```

진짜 Crashlytics 에러 전송을 테스트할 필요가 생기면 그건 Integration Test에서 실기기로 확인하는 게 맞다. 단위 테스트에서 Crashlytics를 테스트한다는 건 "에러 로깅 서버가 동작하는지 단위 테스트한다"는 말인데, 그건 Firebase 팀이 이미 했다.

## CI에서 Firebase 없이 돌리기

이 구조가 잡히면 CI 설정이 단순해진다. `flutter test`만 해도 된다.

```yaml
# .github/workflows/test.yml
- name: Run unit tests
  run: flutter test --coverage
  # Firebase 관련 설정 불필요
  # google-services.json 불필요
  # GOOGLE_APPLICATION_CREDENTIALS 불필요
```

반면 Firebase를 직접 초기화하는 코드가 테스트에 섞여 있으면 `google-services.json`을 CI 시크릿으로 관리하고, 에뮬레이터를 띄우거나 Firebase Emulator Suite를 세팅해야 한다. 작은 팀에서 이걸 유지하는 건 부담이 크다.

격리가 목적이라면 Fake로 가는 게 맞다.

## 한 가지 주의 사항

인터페이스를 뽑아낼 때 Firebase SDK 메서드를 전부 노출하려는 충동이 생긴다. 그러면 안 된다.

```dart
// 나쁜 예 — Firebase API를 그대로 노출
abstract class IRemoteConfigService {
  RemoteConfigValue getValue(String key); // Firebase 타입이 인터페이스에 노출됨
}

// 좋은 예 — 앱이 실제로 쓰는 것만 노출
abstract class IRemoteConfigService {
  bool getBool(String key);
  double getDouble(String key);
  String getString(String key);
}
```

Firebase 구체 타입이 인터페이스에 새어 나오면 Fake 만들기가 힘들어지고, 나중에 Firebase를 다른 걸로 교체할 때도 인터페이스까지 같이 바꿔야 한다.

---

다음 편은 **테스트 커버리지 뱃지 자동화 — Codecov 연동과 PR 커버리지 리포트**를 다룬다. 커버리지 숫자를 어떻게 팀에 보여줄지, PR마다 diff 커버리지를 어떻게 체크하는지 정리할 예정이다.

**참고 링크**
- [FlutterFire Testing 공식 문서](https://firebase.flutter.dev/docs/testing/testing/)
- [firebase_auth_mocks 패키지](https://pub.dev/packages/firebase_auth_mocks)
- [fake_cloud_firestore 패키지](https://pub.dev/packages/fake_cloud_firestore)
