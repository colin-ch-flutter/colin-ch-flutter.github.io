---
layout: post
title: "Flutter 테스트 코드 안티패턴 — 리뷰에서 자주 걸리는 실수들"
description: "Flutter 테스트 리뷰에서 반복적으로 나오는 안티패턴 8가지를 IoT 앱 코드 예시와 함께 정리했다. 슬리피 테스트, Mock 남용, 단일 테스트에 Assert 무더기 넣기 — 이 실수들이 테스트를 느리고 신뢰할 수 없게 만든다."
date: 2026-06-28
tags: [Flutter, Dart, CleanArchitecture, IoT, UX]
comments: true
share: true
---

# Flutter 테스트 코드 안티패턴 — 리뷰에서 자주 걸리는 실수들

![코드 리뷰와 테스트 품질 관리](https://images.unsplash.com/photo-1555949963-aa79dcee981c?w=800&q=80)

테스트를 잘 짜는 것과 테스트가 많은 것은 다르다. IoT 스마트홈 앱 시리즈 내내 테스트를 쌓아왔는데, 어느 시점부터 테스트가 오히려 짐이 되기 시작했다. PR 리뷰마다 같은 코멘트가 반복됐고, 수정하면 전혀 관계없는 테스트가 깨졌다. 원인을 하나씩 찾아보니 전형적인 안티패턴들이었다.

## 1. 슬리피 테스트 — `Future.delayed`로 타이밍 맞추기

가장 자주 보이고, 가장 빨리 팀 전체를 불행하게 만드는 패턴이다.

```dart
// 나쁜 예
test('MQTT 메시지 처리 후 상태 업데이트', () async {
  mqttService.simulateMessage('boiler/status', '{"power":true}');
  await Future.delayed(const Duration(milliseconds: 500)); // 🚨
  expect(controller.isPowerOn, isTrue);
});
```

500ms가 충분할 거라는 가정 위에 쌓인 테스트다. CI 서버가 느리면 실패하고, 로컬에서는 통과한다. 실패할 때도 왜 실패하는지 알 수 없다. `StreamController`와 마이크로태스크 비우기로 대체하면 된다.

```dart
// 좋은 예
test('MQTT 메시지 처리 후 상태 업데이트', () async {
  final statusStream = StreamController<String>();
  when(() => mockMqttService.statusStream)
      .thenAnswer((_) => statusStream.stream);

  statusStream.add('{"power":true}');
  await Future.microtask(() {}); // 마이크로태스크 큐 비우기

  expect(controller.isPowerOn, isTrue);
});
```

`Future.delayed`가 있는 테스트는 PR을 올리기 전에 반드시 지운다. 그게 기준이다.

## 2. Assert 무더기 — 한 테스트에 모든 걸 검증하기

```dart
// 나쁜 예
test('보일러 컨트롤러 초기화', () {
  expect(controller.isPowerOn, isFalse);
  expect(controller.temperature, equals(0));
  expect(controller.mode, equals(BoilerMode.off));
  expect(controller.schedule, isEmpty);
  expect(controller.isConnected, isFalse);
  expect(controller.errorMessage, isNull);
  expect(controller.lastUpdated, isNull);
  // ... 이런 게 12개 더
});
```

테스트가 실패하면 첫 번째 `expect`에서 멈춘다. 나머지 11개가 어떤 상태인지 모른다. 무엇이 진짜 문제인지 파악하려면 디버거를 열어야 한다.

테스트 하나는 하나의 동작만 검증한다. `group`으로 묶으면 가독성도 유지된다.

```dart
// 좋은 예
group('보일러 컨트롤러 초기 상태', () {
  test('전원은 꺼진 상태로 시작한다', () {
    expect(controller.isPowerOn, isFalse);
  });

  test('온도는 0으로 시작한다', () {
    expect(controller.temperature, equals(0));
  });

  test('에러 메시지는 없다', () {
    expect(controller.errorMessage, isNull);
  });
});
```

테스트 케이스 수가 늘어나는 게 아니라 가시성이 좋아지는 거다.

## 3. Mock 남용 — 진짜 객체를 써도 되는데 Mock으로 만들기

```dart
// 나쁜 예
final mockDateTime = MockDateTime();   // DateTime은 값 객체
final mockList = MockList<String>();   // 그냥 [] 쓰면 됨
final mockString = MockString();       // 이건 진짜 심각한 경우
```

[테스트 더블 패턴]({% post_url 2026-06-26-12-00-00-765390-flutter-test-doubles-fake-mock-stub-spy-patterns %})에서 정리한 기준을 그대로 적용하면 된다. Mock은 외부 의존성(네트워크, BLE, DB)에만 쓴다. 값 객체나 컬렉션을 Mock으로 만들면 구현 세부사항에 테스트가 결합된다. 리팩터링할 때마다 Mock 설정이 깨진다.

진짜 객체를 쓸 수 있으면 진짜 객체를 쓴다. 단순한 원칙이다.

## 4. 매직 넘버 — 맥락 없는 숫자

```dart
// 나쁜 예
test('온도 임계값 초과 시 경고 알림', () async {
  await controller.updateTemperature(45); // 45가 왜 45인가
  expect(controller.hasAlert, isTrue);
});
```

6개월 뒤에 이 테스트를 보면 45가 왜 45인지 모른다. 임계값이 43으로 바뀌면 이 테스트도 수정해야 하는지조차 불분명하다. 도메인 상수로 빼면 의도가 명확해진다.

```dart
// 좋은 예
const kBoilerOverheatThreshold = 43; // 제품 스펙: 43도 초과 시 과열 경고

test('보일러 과열 임계값 초과 시 경고 알림이 발생한다', () async {
  await controller.updateTemperature(kBoilerOverheatThreshold + 2);
  expect(controller.hasAlert, isTrue);
});
```

도메인 상수는 테스트 파일 맨 위에 모아두거나, `test_constants.dart`로 분리한다.

## 5. 복붙 테스트 — 셋업 코드 반복

```dart
// 나쁜 예
test('BLE 연결 성공 시 상태 업데이트', () async {
  final fakeBle = FakeBleService();
  final controller = BleController(bleService: fakeBle);
  await controller.onInit();
  fakeBle.simulateConnect();
  expect(controller.connectionState, equals(BleConnectionState.connected));
});

test('BLE 연결 해제 시 상태 업데이트', () async {
  final fakeBle = FakeBleService(); // 동일
  final controller = BleController(bleService: fakeBle); // 동일
  await controller.onInit(); // 동일
  fakeBle.simulateConnect(); // 동일
  fakeBle.simulateDisconnect();
  expect(controller.connectionState, equals(BleConnectionState.disconnected));
});
```

셋업 코드가 3곳 이상 반복되면 `setUp`으로 올린다. [테스트 공유 픽스처 패턴]({% post_url 2026-06-26-18-00-00-802425-flutter-test-refactoring-shared-fixture-helper %})에서 자세히 다뤘는데, 실제로 적용하면 테스트 파일 길이가 절반으로 줄었다.

## 6. 구현 테스트 — 내부 메서드를 직접 검증하기

```dart
// 나쁜 예 — private 메서드 우회 시도
// Dart에서 _로 시작하는 메서드는 라이브러리 외부 접근 불가
// 이걸 하려고 파일 분리하거나 리플렉션을 쓴다면 이미 틀린 방향
test('_parseBoilerStatus 파싱 결과 확인', () {
  // ...
});
```

`_parseBoilerStatus`가 private인 이유가 있다. 구현 세부사항이기 때문이다. 이걸 직접 테스트하면 메서드 이름을 바꾸거나 내부 구조를 바꿀 때마다 테스트가 깨진다. 리팩터링이 불가능해진다.

공개 인터페이스를 통해 결과를 검증하면 충분하다.

```dart
// 좋은 예 — 공개 인터페이스로 검증
test('MQTT 메시지에서 보일러 상태를 정확히 읽어온다', () async {
  mqttFake.emit('boiler/status', '{"power":true,"temperature":22}');
  await controller.refresh();

  expect(controller.isPowerOn, isTrue);
  expect(controller.temperature, equals(22));
});
```

내부가 어떻게 파싱하든 결과가 맞으면 된다. 이게 테스트의 역할이다.

## 7. 조건부 테스트 로직

```dart
// 나쁜 예
test('플랫폼별 BLE 초기화', () async {
  if (Platform.isAndroid) {
    await controller.connectBle();
    expect(controller.isConnected, isTrue);
  } else {
    expect(controller.isConnected, isFalse);
  }
});
```

테스트 안에 `if`가 있으면 사실상 두 개의 테스트가 하나로 뭉쳐진 것이다. 로컬 Mac에서 실행하면 `else` 분기만 실행되고, CI(Ubuntu)에서는 다른 분기가 실행된다. 플랫폼별로 명확히 나눠야 한다.

```dart
// 좋은 예
group('Android BLE 초기화', () {
  test('연결 요청 후 connected 상태가 된다', () async {
    // Android 전용 케이스
  });
}, skip: !Platform.isAndroid);

group('iOS BLE 초기화', () {
  test('시뮬레이터에서는 BLE를 지원하지 않는다', () {
    // iOS 전용 케이스
  });
}, skip: !Platform.isIOS);
```

플랫폼 의존성 자체를 추상화하면 조건부 없이 깔끔하게 처리된다.

## 8. 테스트 이름이 코드인 경우

```dart
// 나쁜 예
test('test1', () { ... });
test('BLE', () { ... });
test('연결 테스트', () { ... });
test('에러 처리', () { ... });
```

3개월 뒤 CI에서 "연결 테스트 FAILED"를 받으면 어디서부터 봐야 할지 모른다. 어떤 연결인지, 어떤 상황인지, 무엇을 기대했는지가 이름에 없다.

```dart
// 좋은 예
test('BLE 기기가 범위를 벗어난 후 재연결 요청하면 자동으로 재시도한다', () { ... });
test('MQTT 브로커 응답이 5초 이내 오지 않으면 TimeoutException을 던진다', () { ... });
test('보일러 전원 끄기 실패 시 에러 메시지를 유지한다', () { ... });
```

"상황 → 행동 → 예상 결과" 구조로 쓰면 테스트 이름 자체가 스펙 문서가 된다.

---

![코드 품질과 테스트 전략 시각화](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

안티패턴을 정리하고 나니 공통점이 보인다. 모두 **빠르게 작성**하려다 생긴 것들이다. 슬리피 테스트는 타이밍을 제대로 다루기 귀찮아서, Assert 무더기는 케이스를 나누기 귀찮아서, 복붙 테스트는 픽스처 설계가 귀찮아서 생긴다.

테스트 품질이 낮으면 실행 속도가 느려지고, 실패해도 신뢰가 없고, 결국 아무도 안 읽는 코드가 된다. 커버리지 숫자는 높은데 실제로는 아무것도 보호하지 않는 상태. 이 시리즈를 진행하면서 가장 자주 되돌아간 게 이 안티패턴 목록이었다.

다음 편은 Flutter Unit Test 시리즈 전체를 돌아보는 **회고**다. 어떤 테스트가 실제로 버그를 잡았고, 어떤 테스트가 오히려 방해가 됐는지 정리할 예정이다.

**참고 링크**
- [Test Anti-patterns — Martin Fowler](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Flutter 공식 Testing 문서](https://docs.flutter.dev/testing)
- [Navigating the Hard Parts of Testing in Flutter — DCM](https://dcm.dev/blog/2025/07/30/navigating-hard-parts-testing-flutter-developers/)
