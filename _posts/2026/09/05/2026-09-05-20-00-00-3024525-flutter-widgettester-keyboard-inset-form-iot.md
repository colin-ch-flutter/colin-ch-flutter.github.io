---
layout: post
title: "Flutter WidgetTester 키보드 테스트 - viewInsets로 IoT 설정 폼 검증하기"
description: "Flutter WidgetTester에서 키보드가 올라온 뒤 입력창이 가려지는 문제를 viewInsets와 ensureVisible로 재현하고, 스마트홈 설정 폼의 검증·스크롤 동작을 안정적으로 테스트하는 방법을 정리했다."
date: 2026-09-05
tags: [Flutter, Dart, 테스트, WidgetTester, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter WidgetTester로 키보드가 열린 IoT 설정 폼 테스트](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Flutter WidgetTester에서 `enterText()`만 호출하면 키보드가 올라온 화면까지 검증한 것처럼 보인다. 실제로는 그렇지 않다. 스마트홈 앱의 Wi-Fi 비밀번호나 기기 별칭 입력창은 키보드에 가려지고, 제출 버튼은 화면 아래로 밀린다. 이번에는 `viewInsets`를 주입하고 `ensureVisible()`로 입력창을 노출시킨 뒤 폼 검증까지 확인했다.

## 입력 테스트만으로는 부족했던 이유

처음에는 아래처럼 텍스트만 넣고 통과를 확인했다.

```dart
await tester.enterText(
  find.byKey(const Key('wifi-password-field')),
  'boiler-1234',
);
expect(find.text('boiler-1234'), findsOneWidget);
```

이 테스트는 값이 들어갔는지만 본다. 키보드가 열린 상태에서 저장 버튼이 보이는지, 빈 값일 때 오류가 나타나는지는 보장하지 않는다.

## `viewInsets`로 키보드가 열린 환경 만들기

WidgetTester의 화면 크기와 키보드 영역을 직접 바꾼다. `viewInsets.bottom`을 320으로 주면 화면 아래 320픽셀이 키보드에 가려진 상황을 흉내 낼 수 있다.

```dart
testWidgets('키보드가 열려도 Wi-Fi 비밀번호 필드가 노출된다', (tester) async {
  await tester.pumpWidget(const TestApp(child: WifiSettingsScreen()));
  await tester.pump();

  tester.view
    ..physicalSize = const Size(1080, 1920)
    ..devicePixelRatio = 2.0
    ..viewInsets = const EdgeInsets.only(bottom: 320);
  addTearDown(tester.view.reset);

  final password = find.byKey(const Key('wifi-password-field'));
  await tester.tap(password);
  await tester.pump();

  await tester.ensureVisible(password);
  await tester.pump();

  expect(tester.widget<TextField>(password).focusNode?.hasFocus, isTrue);
});
```

프로젝트 버전에 따라 접근 API가 달라질 수 있다. 중요한 건 테스트마다 `tester.view.reset`을 호출하는 것이다. 이 정리를 빼먹으면 다음 테스트에 화면 크기와 inset이 남는다.

## `ensureVisible`은 스크롤 완료를 대신하지 않는다

`ensureVisible()` 호출 직후 레이아웃이 바로 확정된다고 가정하면 간헐적으로 실패한다. 한 프레임을 진행한 뒤 저장 버튼의 노출 여부와 검증 문구를 따로 확인한다.

```dart
testWidgets('빈 비밀번호는 오류를 표시하고 저장하지 않는다', (tester) async {
  final repository = FakeDeviceRepository();
  await tester.pumpWidget(TestApp(
    repository: repository,
    child: const WifiSettingsScreen(),
  ));

  await tester.tap(find.byKey(const Key('save-wifi-button')));
  await tester.pump();

  expect(find.text('비밀번호를 입력해 주세요'), findsOneWidget);
  expect(repository.saveCalls, 0);
});
```

여기서 `pumpAndSettle()`을 무조건 쓰지 않았다. 연결 상태 스트림이 살아 있으면 settle이 끝나지 않을 수 있다. 포커스 이동은 `pump()` 한 번이면 충분하고, 애니메이션만 필요한 구간에 제한 시간 있는 settle을 쓴다.

| 확인 대상 | 테스트 기준 | 흔한 실수 |
|---|---|---|
| 키보드 노출 | `viewInsets.bottom` 주입 | 실제 키보드가 뜨길 기다림 |
| 입력창 위치 | `ensureVisible()` 후 `pump()` | `enterText()`만 호출 |
| 폼 오류 | 오류 문구와 저장 호출 수 함께 검증 | 화면 텍스트만 확인 |
| 테스트 격리 | `tester.view.reset` 등록 | 다음 테스트에 inset 잔류 |

## 짧게 정리하면

Flutter WidgetTester 폼 테스트는 값 입력만 검증하면 반쪽이다. `viewInsets`로 키보드가 열린 조건을 만들고, `ensureVisible()` 뒤 한 프레임을 진행한 다음 입력창 포커스·오류 문구·Repository 호출을 각각 확인해야 한다. 특히 IoT 설정 화면처럼 하단 버튼과 실시간 상태 스트림이 함께 있는 화면에서는 `pumpAndSettle()`보다 필요한 이벤트만 `pump()`하는 쪽이 실패 원인을 찾기 쉽다.
