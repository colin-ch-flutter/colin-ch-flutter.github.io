---
layout: post
title: "Flutter 테스트 코드 정리법 — 공유 픽스처, TestHelper, Mother 패턴으로 중복 제거"
description: "Flutter IoT 앱에서 테스트가 쌓일수록 setup 코드가 복붙으로 넘쳐난다. 공유 픽스처, TestHelper 클래스, Object Mother 패턴으로 중복을 제거하고 유지보수 가능한 테스트 구조를 만드는 방법을 정리했다."
date: 2026-06-26
tags: [Flutter, Dart, CleanArchitecture, Repository패턴]
comments: true
share: true
---

# Flutter 테스트 코드 정리법 — 공유 픽스처, TestHelper, Mother 패턴으로 중복 제거

![코드 리팩토링 작업 화면](https://images.unsplash.com/photo-1587620962725-abab7fe55159?w=800&q=80)

테스트가 10개 넘어가면 반드시 생기는 문제가 있다. `setUp()`에 똑같은 초기화 코드가 복사되고, `FakeRepository`를 만드는 코드가 파일마다 조금씩 다르게 중복된다. 처음엔 "나중에 정리하지" 했는데 20개 넘으면 손대기 싫어진다. 테스트 리팩토링을 안 하면 커버리지가 높아도 테스트 코드 자체가 짐이 된다.

## 문제 상황: 복붙 setUp이 쌓이기 시작했다

[이전 편]({% post_url 2026-06-26-16-00-00-790080-flutter-test-coverage-strategy-threshold %})에서 레이어별 커버리지 목표를 잡고 나서 테스트를 쭉 추가했더니, 파일마다 이런 코드가 반복됐다.

```dart
// boiler_controller_test.dart
late FakeBoilerRepository fakeBoilerRepo;
late FakeAuthRepository fakeAuthRepo;
late BoilerController controller;

setUp(() {
  fakeBoilerRepo = FakeBoilerRepository();
  fakeAuthRepo = FakeAuthRepository();
  controller = BoilerController(
    boilerRepo: fakeBoilerRepo,
    authRepo: fakeAuthRepo,
  );
});
```

```dart
// ble_controller_test.dart
late FakeBoilerRepository fakeBoilerRepo;
late FakeAuthRepository fakeAuthRepo;
late FakeBleService fakeBle;
late BleController controller;

setUp(() {
  fakeBoilerRepo = FakeBoilerRepository();
  fakeAuthRepo = FakeAuthRepository();
  fakeBle = FakeBleService();
  controller = BleController(
    bleService: fakeBle,
    boilerRepo: fakeBoilerRepo,
  );
});
```

`FakeBoilerRepository`, `FakeAuthRepository`를 만드는 코드가 파일 5개에 걸쳐 퍼져 있었다. 나중에 `FakeBoilerRepository` 생성자에 파라미터를 추가했더니 5개 파일을 다 고쳐야 했다. 30분짜리 삽질이 이렇게 시작된다.

## 해결 방법 1: TestHelper로 의존성 생성 중앙화

Fake 객체 생성을 한 곳에서 관리하는 `TestHelper` 클래스를 만든다.

```dart
// test/helpers/test_helper.dart
class TestHelper {
  static FakeBoilerRepository boilerRepo() => FakeBoilerRepository();
  static FakeAuthRepository authRepo() => FakeAuthRepository();
  static FakeBleService bleService() => FakeBleService();
  static FakeMqttService mqttService() => FakeMqttService();

  static BoilerController boilerController({
    FakeBoilerRepository? boilerRepo,
    FakeAuthRepository? authRepo,
  }) =>
      BoilerController(
        boilerRepo: boilerRepo ?? TestHelper.boilerRepo(),
        authRepo: authRepo ?? TestHelper.authRepo(),
      );

  static BleController bleController({
    FakeBleService? bleService,
    FakeBoilerRepository? boilerRepo,
  }) =>
      BleController(
        bleService: bleService ?? TestHelper.bleService(),
        boilerRepo: boilerRepo ?? TestHelper.boilerRepo(),
      );
}
```

이렇게 하면 각 테스트 파일에서 setUp이 이렇게 줄어든다.

```dart
setUp(() {
  fakeBoilerRepo = TestHelper.boilerRepo();
  controller = TestHelper.boilerController(boilerRepo: fakeBoilerRepo);
});
```

`FakeBoilerRepository` 생성자가 바뀌면 `TestHelper` 한 곳만 고치면 된다. 처음 이걸 만들고 나서 "이걸 왜 이제 했나" 싶었다.

## 해결 방법 2: Object Mother 패턴으로 테스트 데이터 통일

Fake 객체를 만드는 것 외에 테스트 데이터(Entity, DTO)를 만드는 중복도 심각하다.

```dart
// device_controller_test.dart에서 반복되던 코드
final device = BoilerDevice(
  id: 'device-001',
  name: '보일러 1',
  isOnline: true,
  temperature: 60.0,
  targetTemperature: 65.0,
  mode: BoilerMode.heat,
);
```

이걸 `BoilerDeviceMother`로 뽑아낸다.

```dart
// test/mothers/boiler_device_mother.dart
class BoilerDeviceMother {
  static BoilerDevice online({
    String id = 'device-001',
    String name = '보일러 1',
    double temperature = 60.0,
    double targetTemperature = 65.0,
  }) =>
      BoilerDevice(
        id: id,
        name: name,
        isOnline: true,
        temperature: temperature,
        targetTemperature: targetTemperature,
        mode: BoilerMode.heat,
      );

  static BoilerDevice offline({String id = 'device-001'}) =>
      BoilerDevice(
        id: id,
        name: '오프라인 기기',
        isOnline: false,
        temperature: 0,
        targetTemperature: 0,
        mode: BoilerMode.off,
      );

  static BoilerDevice withError() =>
      BoilerDevice(
        id: 'error-device',
        name: '오류 기기',
        isOnline: false,
        temperature: -1,
        targetTemperature: -1,
        mode: BoilerMode.unknown,
      );
}
```

테스트에서 쓸 때는 이렇게 된다.

```dart
test('오프라인 기기는 제어 버튼을 비활성화해야 한다', () {
  final device = BoilerDeviceMother.offline();
  fakeBoilerRepo.setDevice(device);
  // ...
});
```

`BoilerDeviceMother.online()`, `.offline()`, `.withError()` — 이름만 봐도 어떤 상태를 테스트하는지 알 수 있다. 이게 Object Mother 패턴의 핵심이다.

![테스트 코드 구조 정리](https://images.unsplash.com/photo-1516116216624-53e697fedbea?w=800&q=80)

## 해결 방법 3: 공유 픽스처로 setUp 코드 묶기

같은 레이어의 테스트끼리 setUp 패턴이 동일할 때는 shared fixture로 뽑아낼 수 있다.

```dart
// test/fixtures/boiler_controller_fixture.dart
class BoilerControllerFixture {
  late FakeBoilerRepository boilerRepo;
  late FakeAuthRepository authRepo;
  late BoilerController controller;

  void setUp() {
    boilerRepo = TestHelper.boilerRepo();
    authRepo = TestHelper.authRepo();
    controller = TestHelper.boilerController(
      boilerRepo: boilerRepo,
      authRepo: authRepo,
    );
  }
}
```

테스트 파일에서는 이걸 mixin처럼 가져다 쓴다.

```dart
void main() {
  final fixture = BoilerControllerFixture();

  setUp(() => fixture.setUp());

  test('setTemperature는 Repo에 올바른 값을 전달해야 한다', () async {
    fixture.boilerRepo.setDevice(BoilerDeviceMother.online());
    await fixture.controller.setTemperature(70.0);

    expect(fixture.boilerRepo.lastSetTemperature, 70.0);
  });
}
```

이 구조에서는 `BoilerController`를 쓰는 테스트가 10개든 20개든 setUp 코드가 fixture에 한 번만 있다.

## 디렉터리 구조

```
test/
  helpers/
    test_helper.dart        # Fake 객체 생성 중앙화
  mothers/
    boiler_device_mother.dart
    mqtt_message_mother.dart
    auth_token_mother.dart
  fixtures/
    boiler_controller_fixture.dart
    ble_controller_fixture.dart
  unit/
    boiler_controller_test.dart
    ble_controller_test.dart
  widget/
    boiler_card_test.dart
  integration/
    ...
```

`helpers/`, `mothers/`, `fixtures/`가 테스트 인프라 계층이다. 이 세 개만 잘 관리하면 나머지 테스트 파일들은 "무엇을 테스트하는가"만 담게 된다.

## 삽질 포인트: Mother에서 기본값 선택을 잘못하면 오히려 혼란

`BoilerDeviceMother.online()` 기본 온도를 `60.0`으로 설정했는데, 나중에 "온도 유효성 검사" 테스트에서 `60.0`이 경계값이라 테스트가 실패했다. Mother의 기본값이 테스트 케이스 경계값과 겹치면 안 된다.

규칙을 하나 만들었다: **Mother 기본값은 절대 경계값이 아닌 "명백히 정상인 값"만 쓴다.** 온도 유효 범위가 30~90이면 Mother 기본값은 60으로 하고, 경계 테스트(30, 90, 29, 91)는 직접 넘긴다.

---

다음 편은 [테스트 더블 패턴]({% post_url 2026-06-26-12-00-00-765390-flutter-test-doubles-fake-mock-stub-spy-patterns %})과 이번 리팩토링 구조를 결합해서 **실제 IoT 앱 전체 테스트 전략을 정리하는 회고 편**으로 마무리할 예정이다.

**참고 링크**
- [Flutter testing 공식 문서](https://docs.flutter.dev/testing/overview)
- [Object Mother Pattern](https://martinfowler.com/bliki/ObjectMother.html)
- [Mocktail GitHub](https://github.com/felangel/mocktail)
