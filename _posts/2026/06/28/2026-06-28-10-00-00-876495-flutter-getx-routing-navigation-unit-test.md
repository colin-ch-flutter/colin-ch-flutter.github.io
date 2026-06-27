---
layout: post
title: "GetX 라우팅 단위 테스트 — 화면 전환 로직을 Flutter 없이 검증하는 법"
description: "GetX의 Get.to, Get.off, GetMiddleware를 단위 테스트에서 검증하는 방법을 실제 IoT 앱 시나리오 코드로 정리했다. testMode 설정과 Get.reset() 패턴으로 Flutter 위젯 없이 라우팅 로직만 격리해 테스트한다."
date: 2026-06-28
tags: [Flutter, GetX, CleanArchitecture, IoT, Android, iOS]
comments: true
share: true
---

# GetX 라우팅 단위 테스트 — 화면 전환 로직을 Flutter 없이 검증하는 법

![GetX 라우팅 단위 테스트 Flutter 코드 화면](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

GetX 라우팅은 단위 테스트에서 `Get.testMode = true` 한 줄로 격리할 수 있다. BLE 연결 성공 → 디바이스 화면 이동, 토큰 만료 → 로그인 리다이렉트 같은 시나리오를 위젯 한 줄 없이 검증 가능하다. 문서에는 잘 안 나와 있는 내용인데, 한번 패턴을 잡아두면 나머지 화면 전환 테스트는 거의 복붙 수준이다.

## 왜 라우팅 테스트가 필요한가

시리즈 2편에서 GetX 비동기 다이얼로그 테스트를 다뤘는데, 그때 이미 `Get.testMode`를 언급했다. 라우팅도 같은 원리다.

문제는 이거다. 스마트홈 앱에서 BLE 연결 성공 후 `HomeController.onConnected()`가 `Get.off(DeviceScreen)`를 호출하는 코드가 있다. 이걸 테스트하려면:

- **Widget Test**: `GetMaterialApp`을 띄우고, FakeBleService 주입하고, 연결 흐름 시뮬레이션 후 `DeviceScreen`이 렌더링됐는지 확인 — 세팅만 30줄
- **Unit Test**: `Get.testMode = true` 설정 후 컨트롤러 메서드 호출, `Get.currentRoute` 확인 — 10줄 이내

Widget Test가 필요한 경우도 있지만, **"연결 성공 시 올바른 라우트로 이동하는가"** 는 순수 로직 검증이다. 단위 테스트로 충분하다.

## GetX 테스트 환경 세팅

GetX 라우팅 단위 테스트의 핵심은 두 가지다.

```dart
// test 파일 상단
setUp(() {
  Get.testMode = true;  // 실제 라우팅 대신 내부 상태만 추적
});

tearDown(() {
  Get.reset();  // 테스트 간 상태 완전 초기화
});
```

`Get.testMode = true`를 설정하면 `Get.to()`, `Get.off()` 호출 시 실제 Flutter 위젯 스택을 바꾸지 않고 내부 라우트 기록만 남긴다. `Get.reset()`은 등록된 컨트롤러, 바인딩, 라우트 기록을 전부 날린다. tearDown마다 반드시 호출해야 한다. 안 하면 테스트 순서에 따라 상태가 오염돼서 "혼자 실행하면 통과, 전체 실행하면 실패" 상황이 생긴다.

pubspec.yaml에 특별히 추가할 건 없다. `get` 패키지만 있으면 된다.

## Get.to / Get.off 검증

BleController가 연결 성공 시 디바이스 화면으로 이동하는 시나리오다.

```dart
// lib/controllers/ble_controller.dart
class BleController extends GetxController {
  final BleService _bleService;
  BleController(this._bleService);

  Future<void> connectDevice(String deviceId) async {
    final connected = await _bleService.connect(deviceId);
    if (connected) {
      Get.off(() => DeviceScreen(), routeName: '/device');
    } else {
      Get.snackbar('오류', 'BLE 연결 실패');
    }
  }
}
```

```dart
// test/controllers/ble_controller_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:get/get.dart';

class FakeBleService extends BleService {
  bool shouldSucceed;
  FakeBleService({required this.shouldSucceed});

  @override
  Future<bool> connect(String deviceId) async => shouldSucceed;
}

void main() {
  late BleController controller;

  setUp(() {
    Get.testMode = true;
  });

  tearDown(() {
    Get.reset();
  });

  test('BLE 연결 성공 시 /device 라우트로 이동한다', () async {
    controller = BleController(FakeBleService(shouldSucceed: true));

    await controller.connectDevice('AA:BB:CC:DD:EE:FF');

    // Get.testMode에서는 currentRoute에 마지막으로 이동한 라우트명이 기록된다
    expect(Get.currentRoute, '/device');
  });

  test('BLE 연결 실패 시 라우트 변경 없다', () async {
    controller = BleController(FakeBleService(shouldSucceed: false));
    final initialRoute = Get.currentRoute;

    await controller.connectDevice('AA:BB:CC:DD:EE:FF');

    expect(Get.currentRoute, initialRoute);
  });
}
```

`Get.currentRoute`가 핵심이다. testMode에서는 라우트 이동 시 이 값이 갱신된다.

## GetMiddleware — 인증 게이트 테스트

실제로 더 까다로웠던 건 미들웨어 테스트였다. 토큰이 없으면 `/login`으로 리다이렉트하는 `AuthMiddleware`인데, 이걸 단위 테스트하는 방법이 문서에 거의 없었다.

GetMiddleware의 `redirect()` 메서드는 `RouteSettings?`를 반환한다. null이면 통과, RouteSettings 반환하면 해당 라우트로 리다이렉트. 이 반환값을 직접 assert하면 된다.

```dart
// lib/middleware/auth_middleware.dart
class AuthMiddleware extends GetMiddleware {
  final AuthRepository _authRepository;
  AuthMiddleware(this._authRepository);

  @override
  RouteSettings? redirect(String? route) {
    if (!_authRepository.isLoggedIn) {
      return const RouteSettings(name: '/login');
    }
    return null;
  }
}
```

```dart
// test/middleware/auth_middleware_test.dart
class FakeAuthRepository implements AuthRepository {
  bool isLoggedIn;
  FakeAuthRepository({required this.isLoggedIn});
}

void main() {
  test('로그인 상태가 아니면 /login으로 리다이렉트한다', () {
    final middleware = AuthMiddleware(
      FakeAuthRepository(isLoggedIn: false),
    );

    final result = middleware.redirect('/home');

    expect(result?.name, '/login');
  });

  test('로그인 상태면 리다이렉트 없이 null 반환한다', () {
    final middleware = AuthMiddleware(
      FakeAuthRepository(isLoggedIn: true),
    );

    final result = middleware.redirect('/home');

    expect(result, isNull);
  });
}
```

GetMiddleware의 `redirect()`는 Flutter 라우트 스택과 완전히 무관하게 동작하는 순수 메서드라서, `Get.testMode`도 필요 없다. 그냥 인스턴스 만들어서 메서드 호출하면 된다.

## 실제 IoT 앱 시나리오 — 전체 흐름 검증

![Flutter GetX 네비게이션 흐름 테스트 시나리오](https://images.unsplash.com/photo-1512941937938-a272e4779a44?w=800&q=80)

스마트홈 앱의 실제 흐름을 테스트한 코드다. 로그아웃 → 모든 화면 스택 제거 → 로그인 화면으로 이동하는 시나리오.

```dart
// lib/controllers/auth_controller.dart
class AuthController extends GetxController {
  final AuthRepository _authRepository;
  AuthController(this._authRepository);

  Future<void> logout() async {
    await _authRepository.clearToken();
    Get.offAllNamed('/login');  // 스택 전체 제거 후 로그인으로
  }
}
```

```dart
// test/controllers/auth_controller_test.dart
class FakeAuthRepository implements AuthRepository {
  bool tokenCleared = false;

  @override
  Future<void> clearToken() async {
    tokenCleared = true;
  }

  @override
  bool get isLoggedIn => !tokenCleared;
}

void main() {
  setUp(() {
    Get.testMode = true;
  });

  tearDown(() {
    Get.reset();
  });

  test('로그아웃 시 토큰 삭제 후 /login으로 이동한다', () async {
    final fakeAuth = FakeAuthRepository();
    final controller = AuthController(fakeAuth);

    await controller.logout();

    expect(fakeAuth.tokenCleared, isTrue);
    expect(Get.currentRoute, '/login');
  });
}
```

`Get.offAllNamed('/login')` 이후 `Get.currentRoute`가 `/login`인지 확인하는 것과 함께, 실제로 토큰 삭제가 일어났는지도 함께 검증한다. 라우팅 테스트와 상태 변경 테스트를 같이 묶을 수 있다.

## 삽질했던 부분

**1. `Get.reset()` 안 하면 테스트 오염**

처음에 `tearDown`에서 `Get.reset()` 빼놨더니, 두 번째 테스트부터 `Get.currentRoute`가 이전 테스트의 라우트를 물고 있었다. "혼자 실행하면 그린인데 `flutter test`로 전체 돌리면 레드" — 이게 원인이었다.

**2. routeName 명시 안 하면 currentRoute가 빈 문자열**

```dart
// 이렇게 하면 currentRoute가 ''
Get.off(() => DeviceScreen());

// 이렇게 해야 '/device'가 잡힌다
Get.off(() => DeviceScreen(), routeName: '/device');
```

위젯 클래스 이름 기반으로 라우트를 쓰면 `Get.currentRoute`에 아무것도 안 잡힌다. `routeName`을 명시하거나, `Get.toNamed('/device')` 방식을 써야 한다.

**3. Get.testMode와 Get.offAll의 조합**

`Get.offAll()`은 testMode에서 스택을 다 날리고 나서 `currentRoute`가 예상과 다르게 나오는 경우가 있다. `Get.offAllNamed()`를 쓰는 게 testMode에서 더 예측 가능했다.

---

이번 편으로 GetX 관련 테스트 주제는 — 컨트롤러 비동기 테스트([2편]({% post_url 2026-06-23-11-07-13-641940-flutter-getx-test-async-dialog-confirm %})), 라우팅 테스트(이번 편) — 까지 다뤘다. 다음 편은 `easy_localization`을 테스트에서 격리하는 방법이다. 다국어 앱에서 번역 키 누락을 자동으로 검출하는 패턴도 같이 본다.

**참고 링크**
- [GetX 공식 문서 — 라우트 관리](https://pub.dev/packages/get#route-management)
- [GetX 상태관리 기초 포스트]({% post_url 2026-05-02-09-21-39-037035-getx-state-management-routing-flutter-iot %})
- [Flutter 테스트 더블 패턴]({% post_url 2026-06-26-12-00-00-765390-flutter-test-doubles-fake-mock-stub-spy-patterns %})
