---
layout: post
title: "Flutter integration_test 다국어·RTL 테스트 - IoT 스마트홈 화면 잘림 잡기"
description: "Flutter integration_test에서 한국어와 아랍어 RTL 환경을 비교해 IoT 스마트홈 기기명·보일러 제어 카드가 잘리거나 순서가 뒤집히는 문제를 검증하는 방법을 정리했다."
date: 2026-08-18
tags: [Flutter, Dart, IoT, 스마트홈, 다국어, UX, CI/CD]
comments: true
share: true
---

![Flutter integration_test 다국어·RTL IoT 화면 격리 테스트](/assets/images/flutter-integration-test-localization-rtl-iot.png)

그림에서 봐야 할 부분은 같은 스마트홈 화면을 언어별 테스트 레인으로 분리해 레이아웃과 이벤트를 따로 확인하는 구조다.

Flutter integration_test에서 한국어 화면만 통과했다고 다국어 UI가 안전한 것은 아니다. IoT 앱은 `거실 보일러`처럼 짧은 한국어 기기명으로 개발하다가, 실제 계정의 긴 영어 이름이나 아랍어 RTL 텍스트에서 카드가 잘리고 버튼 아이콘 순서가 뒤집히는 일이 생긴다. 처음엔 `find.text()`만 바꾸면 된다고 생각했는데, 해보니 테스트 데이터와 화면 방향을 함께 고정해야 재현됐다.

## 문자열이 아니라 의미를 찾는다

번역된 문장을 Finder에 직접 넣으면 언어가 바뀌는 순간 테스트가 전부 깨진다. 제어 버튼에는 고정된 `Key`를 주고, 사용자에게 보이는 문장은 별도로 검증한다.

```dart
testWidgets('언어가 달라도 보일러 제어 카드는 탭할 수 있다', (tester) async {
  await tester.pumpWidget(
    TestApp(
      locale: const Locale('ar'),
      direction: TextDirection.rtl,
      deviceName: 'غرفة المعيشة - التدفئة الرئيسية',
    ),
  );
  await tester.pumpAndSettle();

  final card = find.byKey(const Key('boiler-card'));
  final powerButton = find.byKey(const Key('boiler-power-button'));

  expect(card, findsOneWidget);
  expect(powerButton, findsOneWidget);
  expect(tester.getSemantics(powerButton), matchesSemantics(
    hasTapAction: true,
    isButton: true,
  ));

  await tester.tap(powerButton);
  await tester.pump();
  expect(find.byKey(const Key('command-pending')), findsOneWidget);
});
```

텍스트를 완전히 무시하지는 않는다. 번역 파일의 핵심 문구가 실제로 표시되는지는 언어별 테스트에서 한두 개만 확인하고, 탭 대상·상태·라우팅은 `Key`와 Semantics로 검사했다. 그래야 번역 문구가 다듬어져도 테스트가 의미 없이 깨지지 않는다.

## RTL은 방향만 바꾸면 끝나지 않는다

`Directionality`를 감싸는 것만으로는 부족하다. `Row` 안에서 화살표 아이콘을 직접 회전시켰거나, `EdgeInsets.only(left: 16)`을 곳곳에 사용했다면 RTL에서 여백과 아이콘이 서로 다른 기준으로 움직인다.

| 확인 항목 | 한국어 LTR | 아랍어 RTL | 테스트 기준 |
|---|---|---|---|
| 기기명 | 왼쪽 정렬 | 오른쪽 정렬 | 한 줄 또는 의도한 말줄임 |
| 전원 아이콘 | 카드 오른쪽 | 카드 왼쪽 | 탭 영역은 동일 |
| 온도 증감 | `- 20 +` | 시각 순서 반전 가능 | 의미 순서 유지 |
| 뒤로가기 | 왼쪽 화살표 | 오른쪽 화살표 | 라우트 pop 1회 |

실패했던 코드는 `Text('20°')`의 위치만 찾고 카드 폭을 검사하지 않았다. 긴 아랍어 기기명이 두 줄을 넘어도 테스트는 통과했다. 그래서 레이아웃 계약을 화면 폭 기준으로 직접 확인하는 보조 검증을 넣었다.

```dart
final cardSize = tester.getSize(find.byKey(const Key('boiler-card')));
expect(cardSize.width, greaterThan(240));
expect(find.byKey(const Key('boiler-name-overflow')), findsNothing);
```

문자열 폭은 폰트와 플랫폼에 따라 달라진다. `greaterThan(240)` 같은 절대값을 모든 기기에 강제하면 Firebase Test Lab의 작은 화면에서 또 다른 flaky가 생긴다. 실제 프로젝트에서는 카드가 차지할 수 있는 최대 폭과 `maxLines: 1`, `TextOverflow.ellipsis` 같은 제품 규칙을 먼저 정하고, 테스트는 그 규칙을 확인하는 수준으로 제한했다.

## 테스트 환경을 매번 명시한다

통합 테스트 파일 안에서 전역 locale을 바꾸면 뒤의 테스트가 이전 방향을 물려받는다. 각 시나리오가 앱을 새로 만들고, 끝난 뒤에는 기본 방향으로 돌려놓는 쪽이 안전하다.

```dart
setUp(() async {
  TestWidgetsFlutterBinding.ensureInitialized();
  await testerBinding.setLocale('ko', 'KR');
});

tearDown(() async {
  await testerBinding.setLocale('en', 'US');
});
```

실제 `integration_test`에서는 플랫폼별 locale 변경 API가 달라질 수 있으므로, 앱 내부의 `TestApp(locale: ...)` 주입과 OS locale 테스트를 나눴다. 앱 번역·레이아웃 회귀는 Fake Repository로 빠르게 돌리고, 시스템 언어·키보드·접근성 글자 크기는 Android/iOS 실기기에서 작은 매트릭스로 확인했다.

짧게 정리하면 이렇다.

- 번역 문장보다 `Key`와 Semantics를 안정적인 테스트 계약으로 둔다.
- RTL에서는 정렬, 아이콘 방향, 여백, 탭 영역을 따로 확인한다.
- 긴 기기명과 작은 화면을 넣어야 잘림 버그가 재현된다.
- locale과 `Directionality`는 테스트마다 새로 주입해 상태가 새지 않게 한다.

한국어에서 잘 보이는 Flutter 스마트홈 화면은 출발점일 뿐이다. 실제 사용자의 언어와 기기명 길이를 테스트 데이터로 넣어야 integration_test가 번역 확인을 넘어 화면 품질 회귀 테스트가 된다.
