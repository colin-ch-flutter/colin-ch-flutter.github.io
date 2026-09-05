---
layout: post
title: "Flutter WidgetTester 연속 탭 테스트 - 스마트홈 퀵모드 중복 실행 막기"
description: "Flutter WidgetTester로 스마트홈 퀵모드 버튼의 연속 탭을 재현하고, debounce·isSubmitting 상태와 Repository 호출 횟수를 검증하는 방법을 정리했다."
date: 2026-09-06
tags: [Flutter, Dart, 테스트, WidgetTester, IoT, 스마트홈, UX]
comments: true
share: true
---
![Flutter WidgetTester로 스마트홈 퀵모드 버튼 연속 탭 테스트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter WidgetTester에서 버튼을 한 번 탭했다고 테스트가 끝난 것은 아니다. 스마트홈 앱의 “외출 모드” 버튼은 빠르게 두 번 눌릴 수 있고, 같은 MQTT 명령이 두 번 발행될 수 있다. 화면의 로딩 표시와 함께 Repository 호출 횟수·버튼 잠금 상태를 검사해야 한다.

## 첫 탭 테스트만으로는 부족하다

실제 기기에서 빠르게 두 번 떼면 첫 번째 작업이 끝나기 전에 두 번째 `onTap`이 실행된다. Fake Repository 로그에는 `applyAwayMode()`가 2회 찍혔지만 UI는 한 번만 바뀌어 발견이 늦었다.

| 조건 | 화면 상태 | 명령 호출 |
|---|---|---:|
| 첫 탭 | 적용 중, 버튼 비활성 | 1회 |
| 처리 중 두 번째 탭 | 적용 중 유지 | 1회 |
| 처리 완료 후 탭 | 완료 후 새 실행 | 2회 |

## Fake로 Future 완료 시점을 제어한다

두 번째 탭을 재현하려면 Repository의 응답을 테스트가 직접 완료할 수 있어야 한다.

```dart
class FakeQuickModeRepository implements QuickModeRepository {
  int applyCalls = 0;
  final completer = Completer<void>();

  @override
  Future<void> applyAwayMode() => (++applyCalls, completer.future).$2;
}
```

Controller는 처리 중 재진입을 막는다.

```dart
Future<void> applyAwayMode() async {
  if (isSubmitting.value) return;
  isSubmitting.value = true;
  try { await repository.applyAwayMode(); }
  finally { isSubmitting.value = false; }
}
```

```dart
testWidgets('처리 중 연속 탭은 퀵모드를 중복 실행하지 않는다', (tester) async {
  final fake = FakeQuickModeRepository();
  await tester.pumpWidget(TestApp(repository: fake));

  final button = find.byKey(const ValueKey('away-mode'));
  await tester.tap(button);
  await tester.pump();
  await tester.tap(button);
  await tester.pump();

  expect(fake.applyCalls, 1);
  expect(tester.widget<ElevatedButton>(button).onPressed, isNull);

  fake.completer.complete();
  await tester.pump();
  expect(tester.widget<ElevatedButton>(button).onPressed, isNotNull);
});
```

`pumpAndSettle()` 대신 탭 직후 `pump()`하고 Future를 완료한 뒤 다시 한 프레임만 진행한다.

`Timer` 기반 debounce는 네트워크 요청 중 재진입을 막지 못한다. 버튼은 `isSubmitting`으로 잠그고, 테스트에서는 `onPressed == null`과 Fake 호출 횟수를 함께 검사한다. `finally`에서 플래그를 되돌려 오류 뒤 영구 잠금도 막아야 한다.

스마트홈 버튼은 한 번 성공하는 것만큼, 실수로 두 번 눌러도 한 번만 실행되는지가 품질을 결정한다.
