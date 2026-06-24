---
layout: post
title: "Flutter Widget Test — IoT 제어 화면 UI 테스트 실전"
description: "GetX Controller를 주입해 보일러 제어 화면의 버튼·상태 텍스트를 WidgetTester로 검증하는 방법. pump 타이밍 실수부터 Binding 오류까지 실제로 막혔던 지점을 코드와 함께 정리했다."
date: 2026-06-25
tags: [Flutter, Dart, GetX, CleanArchitecture, IoT]
comments: true
share: true
---

# Flutter Widget Test — IoT 제어 화면 UI 테스트 실전

![Flutter Widget Test 화면 검증](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

Unit Test는 어느 정도 정착이 됐는데, Widget Test는 손을 못 대고 있었다. Repository 레이어, GetX Controller 테스트는 [이전 글에서 다뤘지만]({% post_url 2026-06-23-10-00-00-629595-flutter-unit-test-repository-layer-start %}), 실제 버튼이 눌렸을 때 화면이 어떻게 바뀌는지까지 자동으로 검증하는 건 또 다른 문제다.

IoT 앱에서 Widget Test가 특히 유용한 이유가 있다. 보일러 전원 버튼 누르면 상태 텍스트가 "켜짐"으로 바뀌는지, 연결 상태가 "disconnected"일 때 제어 버튼이 비활성화되는지 — 이런 걸 매번 실기기 켜서 확인하는 건 고역이다. Widget Test로 커버하면 CI에서 잡힌다.

## GetX + Widget Test, 처음엔 막혔다

처음 `testWidgets`에 `BoilerControlScreen()`을 그냥 넣었더니 바로 터졌다.

```
Error: GetxError - "BoilerController" not found. You need to call "Get.put(BoilerController())"
```

GetX는 Widget이 렌더링될 때 Controller를 찾는데, 테스트 환경에는 아무것도 등록이 안 된 상태라 당연한 결과다. Binding을 설정하거나 `Get.put()`으로 직접 주입해줘야 한다.

`Get.put()`이 더 간단하다. 다만 테스트 격리를 위해 **매 테스트마다 `Get.reset()`을 호출해야 한다.** 이걸 빠뜨리면 앞 테스트에서 등록한 Controller가 다음 테스트에 남아서 이상한 상태 오염이 생긴다.

```dart
setUp(() {
  Get.reset(); // 매 테스트 전 GetX 인스턴스 초기화
});
```

## 테스트 대상: 보일러 제어 화면

실제로 테스트할 화면은 이렇게 생겼다.

```dart
class BoilerControlScreen extends GetView<BoilerController> {
  const BoilerControlScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('보일러 제어')),
      body: Obx(() {
        final isConnected = controller.isConnected.value;
        final isPowerOn = controller.isPowerOn.value;

        return Column(
          children: [
            Text(
              isConnected ? '연결됨' : '연결 끊김',
              key: const Key('connectionStatus'),
            ),
            ElevatedButton(
              key: const Key('powerButton'),
              onPressed: isConnected ? controller.togglePower : null,
              child: Text(isPowerOn ? '끄기' : '켜기'),
            ),
          ],
        );
      }),
    );
  }
}
```

`Key`를 명시적으로 달아두는 게 중요하다. `find.text('켜기')`도 작동하긴 하지만, 다국어 지원이나 텍스트 변경 시 테스트까지 같이 깨진다. `find.byKey(Key('powerButton'))` 쪽이 훨씬 안정적이다.

## FakeBoilerController 만들기

실제 `BoilerController`는 BLE/MQTT에 의존하니까 테스트용 Fake를 만든다.

```dart
class FakeBoilerController extends GetxController
    implements BoilerController {
  @override
  final RxBool isConnected = false.obs;
  @override
  final RxBool isPowerOn = false.obs;

  @override
  void togglePower() {
    if (!isConnected.value) return;
    isPowerOn.value = !isPowerOn.value;
  }

  // 테스트에서 상태를 직접 조작하기 위한 헬퍼
  void simulateConnect() => isConnected.value = true;
  void simulateDisconnect() => isConnected.value = false;
}
```

`implements BoilerController`로 타입을 맞춰놓으면 `Get.put<BoilerController>(fake)`로 주입할 때 타입 충돌이 없다.

## Widget Test 작성

```dart
void main() {
  late FakeBoilerController fakeController;

  setUp(() {
    Get.reset();
    fakeController = Get.put<BoilerController>(
      FakeBoilerController(),
    ) as FakeBoilerController;
  });

  Future<void> pumpScreen(WidgetTester tester) async {
    await tester.pumpWidget(
      GetMaterialApp(
        home: const BoilerControlScreen(),
      ),
    );
  }

  testWidgets('연결 끊김 상태에서 전원 버튼이 비활성화된다', (tester) async {
    await pumpScreen(tester);

    // 초기 상태: disconnected
    expect(find.text('연결 끊김'), findsOneWidget);

    final button = tester.widget<ElevatedButton>(
      find.byKey(const Key('powerButton')),
    );
    expect(button.onPressed, isNull); // 비활성화 확인
  });

  testWidgets('연결 후 전원 버튼 누르면 상태가 켜짐으로 바뀐다', (tester) async {
    await pumpScreen(tester);

    fakeController.simulateConnect();
    await tester.pump(); // Obx 리빌드 트리거

    expect(find.text('연결됨'), findsOneWidget);
    expect(find.text('켜기'), findsOneWidget);

    await tester.tap(find.byKey(const Key('powerButton')));
    await tester.pump();

    expect(find.text('끄기'), findsOneWidget);
  });
}
```

### `pump` vs `pumpAndSettle` — 이걸로 한참 헤맸다

처음엔 `pumpAndSettle()`을 모든 곳에 썼다. 근데 MQTT 상태 변화처럼 **주기적으로 업데이트되는 스트림이 물려있으면 `pumpAndSettle`이 타임아웃으로 터진다.** 프레임이 계속 쌓이니까 "안정화"될 타이밍이 없다.

```
// 이렇게 하면 스트림 연결 시 타임아웃 발생
await tester.pumpAndSettle(); // ← 위험

// 이렇게 한 프레임만 처리
await tester.pump(); // ← 안전
```

단순 상태 변화 검증은 `pump()` 한 번이면 충분하다. 애니메이션이 있는 경우에만 `pumpAndSettle()`을 쓰되, 스트림이 없는 환경에서만.

## 여러 상태 조합 테스트

IoT 제어 화면은 상태 조합이 많다. 연결 여부 × 전원 상태 × 오류 상태. 각각 다 케이스를 짜면 좋지만 핵심 흐름 3~4개만 잡아도 실제 회귀를 많이 잡는다.

```dart
testWidgets('연결 중 끊기면 전원 버튼이 즉시 비활성화된다', (tester) async {
  fakeController.simulateConnect();
  fakeController.isPowerOn.value = true;

  await pumpScreen(tester);
  await tester.pump();

  // 연결된 상태 확인
  expect(find.text('연결됨'), findsOneWidget);

  // 갑자기 끊김
  fakeController.simulateDisconnect();
  await tester.pump();

  final button = tester.widget<ElevatedButton>(
    find.byKey(const Key('powerButton')),
  );
  expect(button.onPressed, isNull); // 즉시 비활성화
  expect(find.text('연결 끊김'), findsOneWidget);
});
```

이 케이스가 실제로 버그를 잡았다. MQTT 연결이 끊겼을 때 `isConnected`를 `false`로 업데이트하는 코드가 빠져있었는데, Widget Test 추가하면서 발견했다. 실기기에서는 재연결이 워낙 빨라서 못 보던 거였다.

![Flutter Widget Test 결과 화면](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

## `GetMaterialApp` vs `MaterialApp` — 반드시 전자를 써야 한다

`MaterialApp`으로 감싸면 `GetView`의 Controller 주입이 안 된다. `GetMaterialApp`을 써야 GetX의 라우팅·의존성 주입이 활성화된다. 에러 메시지가 Controller not found로 나오니까 처음엔 Binding 문제인 줄 알았다.

```dart
// 잘못된 것
await tester.pumpWidget(MaterialApp(home: BoilerControlScreen()));

// 맞는 것
await tester.pumpWidget(GetMaterialApp(home: const BoilerControlScreen()));
```

## 정리

Flutter Widget Test를 IoT 제어 화면에 적용할 때 핵심은 세 가지다:
- `Get.reset()` + `Get.put(FakeController())` 조합으로 매 테스트 격리
- `pump()`와 `pumpAndSettle()` 구분 — 스트림이 있으면 `pump()` 써야 타임아웃 안 남
- Widget에 `Key`를 달아두면 `find.byKey()`로 훨씬 안정적인 탐색 가능

다음 편은 이 Widget Test를 [GitHub Actions CI]({% post_url 2026-06-24-16-00-00-678975-flutter-unit-test-github-actions-ci-coverage %})에 통합해서 PR마다 자동으로 돌아가게 만드는 내용이다. Unit Test와 Widget Test를 같은 파이프라인에서 커버리지 합산하는 방법을 다룬다.

---

**참고 링크**
- [Flutter 공식 Widget Test 가이드](https://docs.flutter.dev/cookbook/testing/widget/introduction)
- [Flutter Testing Overview](https://docs.flutter.dev/testing/overview)
- [GetX 공식 문서](https://pub.dev/packages/get)
