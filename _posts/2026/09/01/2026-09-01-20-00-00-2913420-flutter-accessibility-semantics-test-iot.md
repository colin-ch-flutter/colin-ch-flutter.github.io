---
layout: post
title: "Flutter 접근성 테스트 - SemanticsTester로 IoT 제어 카드 검증하기"
description: "Flutter 접근성 테스트에서 SemanticsTester를 사용해 IoT 제어 카드의 label·value·button 상태를 검증하고, TalkBack과 VoiceOver에서 깨지는 접근성 문구를 미리 잡는 방법을 정리했다."
date: 2026-09-01
tags: [Flutter, Dart, 접근성, 테스트, IoT, 스마트홈]
comments: true
share: true
---

![Flutter 접근성 테스트와 IoT 제어 카드 Semantics 트리](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Flutter 접근성 테스트는 화면이 예쁘게 보이는지만 확인하지 않는다. 스크린리더가 보일러 카드를 “보일러, 현재 온도 22도, 연결됨”이라고 읽는지까지 검증해야 한다. 이번에는 `SemanticsTester`로 IoT 제어 카드의 label, value, 버튼 상태를 단위 테스트에서 잡았다.

## 아이콘만 있는 버튼은 테스트 전까지 문제를 모른다

처음 구현할 때 온도 버튼에 아이콘만 넣었다. 손가락으로 누르는 데는 문제가 없었지만 TalkBack에서는 “버튼”이라고만 읽혔다. 연결 상태도 색상으로만 표현했다.

접근성에서 확인할 값은 대략 다음 세 가지다.

| 대상 | 나쁜 결과 | 기대 결과 |
|---|---|---|
| 온도 카드 | 보일러, 22 | 보일러, 현재 온도 22도 |
| 올림 버튼 | 버튼 | 온도 올리기, 버튼 |
| 연결 상태 | 초록색 아이콘 | 연결됨 |

## Semantics를 명시적으로 만든다

카드 전체에는 읽을 문장을 만들고, 아이콘 버튼에는 동작을 label로 붙였다.

```dart
Widget boilerCard({required int temperature, required bool connected}) {
  return MergeSemantics(
    child: Semantics(
      container: true,
      label: '보일러',
      value: '현재 온도 $temperature도',
      hint: connected ? '연결된 상태에서 조절할 수 있음' : '연결이 필요함',
      child: Column(
        children: [
          Text('$temperature°C'),
          Semantics(
            button: true,
            enabled: connected,
            label: '온도 올리기',
            child: IconButton(
              onPressed: connected ? () {} : null,
              icon: const Icon(Icons.keyboard_arrow_up),
            ),
          ),
          Text(connected ? '연결됨' : '연결 끊김'),
        ],
      ),
    ),
  );
}
```

화면에는 섭씨 기호가 맞지만, 음성 출력에서는 “22도”가 더 자연스럽다. `value`를 별도로 지정한 이유다.

## SemanticsTester로 읽기 트리를 검증한다

위젯 테스트에서는 실제 픽셀 대신 접근성 트리에 원하는 문장이 들어갔는지 확인한다.

```dart
testWidgets('연결된 보일러 카드의 접근성 정보가 노출된다', (tester) async {
  final semantics = SemanticsTester(tester);
  addTearDown(semantics.dispose);

  await tester.pumpWidget(
    MaterialApp(home: boilerCard(temperature: 22, connected: true)),
  );

  expect(
    semantics,
    includesNodeWith(
      label: '보일러',
      value: '현재 온도 22도',
      hint: '연결된 상태에서 조절할 수 있음',
    ),
  );
  expect(semantics, includesNodeWith(label: '온도 올리기'));
  expect(semantics, includesNodeWith(label: '연결됨'));
});
```

처음에는 `MergeSemantics` 때문에 카드와 버튼 노드가 예상보다 합쳐져 테스트가 실패했다. 트리를 확인한 뒤 카드 설명과 조작 버튼의 범위를 분리했다.

## 상태별 접근성도 따로 확인한다

연결이 끊겼을 때 버튼을 비활성화했다면 `enabled: false`까지 검사해야 한다. 문구만 맞고 실제로 활성화되어 있으면 스크린리더 사용자는 누를 수 있다고 오해한다.

```dart
testWidgets('연결 끊김 상태는 버튼을 비활성화한다', (tester) async {
  final semantics = SemanticsTester(tester);
  addTearDown(semantics.dispose);
  await tester.pumpWidget(
    MaterialApp(home: boilerCard(temperature: 22, connected: false)),
  );
  expect(semantics,
      includesNodeWith(label: '온도 올리기', enabled: false));
  expect(semantics, includesNodeWith(label: '연결 끊김'));
});
```

## 짧게 정리하면

- 색상과 아이콘만으로 상태를 표현하지 말고 `Semantics` 문장을 함께 만든다.
- `label`은 대상, `value`는 현재 값, `hint`는 사용 방법으로 나누면 읽기 흐름이 안정적이다.
- `SemanticsTester`는 실제 기기 없이도 접근성 트리의 회귀를 잡아준다.
- BLE나 MQTT 상태가 바뀌는 앱일수록 연결됨·연결 끊김·비활성화 상태를 각각 테스트한다.
