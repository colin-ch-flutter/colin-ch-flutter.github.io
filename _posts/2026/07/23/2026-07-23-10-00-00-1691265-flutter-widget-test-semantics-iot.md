---
layout: post
title: "Flutter Widget Test Semantics - IoT 제어 화면 접근성 검증하기"
description: "Flutter Widget Test에서 SemanticsTester로 스마트홈 보일러 버튼의 음성 라벨, 현재 상태, 탭 액션을 검증하는 방법을 실제 IoT 화면 기준으로 정리했다."
date: 2026-07-23
tags: [Flutter, Dart, IoT, 스마트홈, UX]
comments: true
share: true
---

![Flutter Widget Test Semantics로 검증하는 스마트홈 제어 화면](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

핵심은 카드 UI가 아니라 보일러 상태를 보조기술도 읽는다는 점이다.

Flutter Widget Test에서 픽셀과 텍스트만 확인하면 접근성 버그를 놓친다. 아이콘만 있는 전원 버튼에 라벨이 없거나, `켜짐` 상태가 음성 안내에 포함되지 않는 문제는 실제 기기에서야 발견된다. 이번에는 `SemanticsTester`로 이 조건을 테스트에 넣었다.

## 아이콘 버튼이 왜 문제였나

처음에는 `IconButton`에 `tooltip`만 넣으면 충분하다고 생각했다. TalkBack에서는 읽혔지만, 카드 전체를 감싼 `Semantics`와 합쳐지면서 “전원”만 읽히는 경우가 있었다. 상태와 동작을 하나의 의미 단위로 명시해야 했다.

| 검증 대상 | 기대값 | 빠뜨렸을 때 |
|---|---|---|
| `label` | 보일러 전원 | 아이콘 이름만 읽힘 |
| `value` | 켜짐·꺼짐 | 현재 상태를 알 수 없음 |
| `isButton` | 탭 가능한 컨트롤 | 일반 텍스트처럼 안내됨 |
| `tap` action | 탭 액션 존재 | 키보드·스크린 리더 조작 불가 |

화면에서는 상태를 합친 Semantics 노드를 만든다.

```dart
Semantics(
  label: '거실 보일러 전원',
  value: isOn ? '켜짐' : '꺼짐',
  button: true,
  enabled: !isLoading,
  onTap: isLoading ? null : onToggle,
  child: Icon(isOn ? Icons.power : Icons.power_off),
)
```

테스트에서는 화면을 탭하는 것보다 의미 트리를 직접 확인한다. 그래야 글자나 아이콘이 바뀌어도 접근성 계약은 그대로 검증된다.

```dart
testWidgets('보일러 전원 Semantics를 노출한다', (tester) async {
  await tester.pumpWidget(
    MaterialApp(home: BoilerCard(isOn: true, onToggle: () {})),
  );

  final semantics = SemanticsTester(tester);
  expect(
    semantics,
    includesNodeWith(
      label: '거실 보일러 전원',
      value: '켜짐',
      flags: <SemanticsFlag>[SemanticsFlag.isButton],
      actions: <SemanticsAction>[SemanticsAction.tap],
    ),
  );
  semantics.dispose();
});
```

`SemanticsTester`를 만들고 `dispose()`하지 않으면 테스트 뒤에 semantics owner 경고가 남는다. `addTearDown(semantics.dispose)`로 공통 정리해도 된다.

## 로딩 상태도 따로 검증한다

MQTT 명령을 보내는 동안 버튼을 막는다면 `enabled: false`와 `onTap: null`이 함께 반영돼야 한다. 단순히 스피너가 보이는지만 테스트하면 스크린 리더 사용자는 계속 탭 가능한 버튼으로 들을 수 있다.

이 상태는 `SemanticsTester`에서 같은 라벨에 `tap` 액션이 없는지 확인하면 된다. `enabled: false`만 화면에 반영하고 액션을 남겨두면 스크린 리더 사용자는 계속 탭 가능한 버튼으로 듣게 된다.

## 짧게 정리하면

접근성 테스트는 디자인 검수의 부록이 아니다. `SemanticsTester`로 라벨·상태·액션을 고정하면 Flutter 스마트홈 카드 UI를 바꿔도 음성 안내가 깨지는 일을 막을 수 있다.
