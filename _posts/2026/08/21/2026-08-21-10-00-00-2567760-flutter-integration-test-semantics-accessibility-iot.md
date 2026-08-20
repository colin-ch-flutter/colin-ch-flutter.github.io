---
layout: post
title: "Flutter integration_test 접근성 검증 - IoT 제어 카드 Semantics 테스트"
description: "Flutter integration_test에서 IoT 제어 카드의 Semantics 라벨과 스위치 상태를 검증하고, 시각적으로만 통과하는 접근성 버그를 잡는 방법을 정리했다."
date: 2026-08-21
tags: [Flutter, Dart, IoT, 스마트홈, UX, Android, iOS]
comments: true
share: true
---

![Flutter integration_test로 IoT 제어 카드 접근성 검증](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=800&q=80)

이 그림에서 볼 부분은 화면이 예쁘게 보이는 것과 보조 기술이 읽을 수 있는 것은 별개라는 점이다.

Flutter integration_test로 IoT 화면을 검증하면서 놓치기 쉬운 버그가 하나 있었다. 거실 조명 카드는 켜지고 꺼졌지만, VoiceOver가 읽는 문장은 계속 “조명, 스위치”였다. 테스트는 통과했고 실제 동작도 됐다. 하지만 시각장애 사용자는 현재 상태를 알 수 없었다.

## 텍스트를 찾는 테스트만으로는 부족하다

처음에는 `find.text('거실 조명')`으로 카드가 나타났는지만 확인했다. 이 검사는 화면 회귀에는 도움이 되지만 `Semantics` 트리의 라벨과 값까지 보장하지 않는다. 아이콘 버튼에 `tooltip`만 붙여도 화면에는 문제가 없어 보여서 더 늦게 발견됐다.

IoT 제어 카드에는 최소한 아래 세 가지가 하나의 의미 단위로 노출돼야 한다.

| 검증 대상 | 실패하면 생기는 문제 | 테스트 기준 |
|---|---|---|
| 기기 이름 | 어떤 기기인지 알 수 없음 | `거실 조명` |
| 현재 상태 | 켜짐·꺼짐을 구분할 수 없음 | `켜짐` 또는 `꺼짐` |
| 조작 가능 여부 | 버튼인지 인식하지 못함 | `tappable` 또는 스위치 역할 |

## Semantics 정보를 명시적으로 만든다

상태 텍스트를 화면에 그리는 것과 접근성 값으로 제공하는 것을 분리하지 않도록 카드 전체에 의미를 조합했다. 아래 구조는 화면에 보이는 텍스트가 바뀌어도 테스트 대상인 라벨은 일정하게 유지한다.

```dart
Semantics(
  container: true,
  label: device.name,
  value: device.isOn ? '켜짐' : '꺼짐',
  toggled: device.isOn,
  button: true,
  onTap: controller.toggle,
  child: DeviceControlCard(device: device),
)
```

여기서 `toggled`를 빼먹었던 것이 첫 번째 실수였다. `value`는 읽혔지만 스위치가 어떤 상태인지 보조 기술이 일관되게 해석하지 못했다. 또 카드 안쪽의 아이콘 버튼에도 별도 `Semantics`가 남아 있으면 같은 조작이 두 번 노출될 수 있어 `ExcludeSemantics`로 장식용 아이콘을 감쌌다.

## integration_test에서 실제 접근성 트리를 확인한다

테스트에서는 화면에 보이는 문자열보다 semantics 노드의 역할과 상태를 확인한다. `SemanticsTester`는 위젯 테스트에 적합하고, 앱 흐름과 라우팅까지 포함한 이번 검증은 integration_test에서 `Semantics` 노드를 읽는 전용 헬퍼를 두었다.

```dart
testWidgets('거실 조명 상태가 접근성 정보에 반영된다', (tester) async {
  await tester.pumpWidget(const TestApp(initialLightOn: false));
  await tester.pumpAndSettle();

  final before = tester
      .getSemantics(find.bySemanticsLabel('거실 조명'))
      .getSemanticsData();
  expect(before.value, '꺼짐');
  expect(before.hasFlag(SemanticsFlag.hasToggledState), isTrue);

  await tester.tap(find.bySemanticsLabel('거실 조명'));
  await tester.pumpAndSettle();

  final after = tester
      .getSemantics(find.bySemanticsLabel('거실 조명'))
      .getSemanticsData();
  expect(after.value, '켜짐');
});
```

실제 프로젝트에서는 `find.bySemanticsLabel`만 믿지 않고, 동일한 라벨이 여러 공간에 있을 때 `ancestor` 범위를 카드 단위로 제한했다. “조명”이라는 라벨만 찾으면 거실과 침실 카드 중 먼저 발견된 노드를 누르는 문제가 생겼기 때문이다.

## CI에서 확인할 범위

접근성 테스트를 모든 화면에 억지로 넣으면 유지보수 비용이 커진다. 나는 사용자가 직접 조작하는 핵심 흐름만 고정했다.

- 기기 이름·상태·조작 역할이 한 노드에 노출되는가
- MQTT ACK 전후로 `꺼짐 → 켜짐` 값이 바뀌는가
- 로딩·오프라인·권한 거부 상태도 읽을 수 있는가
- 아이콘과 중복 버튼이 하나의 조작으로 합쳐졌는가

솔직하게 정리하면, 접근성 테스트는 별도 기능을 추가하는 일이 아니라 이미 화면에 있는 상태를 사용자에게 정확히 전달하는지 확인하는 일이다. `find.text`가 통과했다고 끝내지 말고, IoT 제어처럼 상태 변화가 많은 화면은 `Semantics` 라벨·값·역할을 함께 고정해야 한다. 그래야 CI가 초록색이어도 실제 사용자는 현재 상태를 모르는 상황을 줄일 수 있다.
