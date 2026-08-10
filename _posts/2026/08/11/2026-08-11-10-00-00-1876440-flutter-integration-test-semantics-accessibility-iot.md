---
layout: post
title: "Flutter integration_test 접근성 테스트 - Semantics로 IoT 제어 화면 검증"
description: "Flutter integration_test와 Semantics 기반 Finder를 사용해 IoT 스마트홈 화면의 버튼 이름, 현재 상태, 탭 가능 여부를 검증하는 방법을 실제 테스트 코드로 정리했다."
date: 2026-08-11
tags: [Flutter, Dart, IoT, 스마트홈, integration_test, UX, Android, iOS]
comments: true
share: true
---

![Flutter integration_test로 스마트홈 접근성 UI를 검증하는 모바일 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter integration_test에서 텍스트가 보이고 탭이 된다는 것만 확인하면 부족하다. 스마트홈 앱에서는 `거실 보일러 켜기` 같은 Semantics 이름과 `켜짐` 상태가 보조공학 도구에 제대로 전달되는지도 검증해야 한다. 이번에는 실제 IoT 제어 화면에서 접근성 정보가 깨지는 문제를 `bySemanticsLabel`로 잡았다.

## 텍스트 테스트만으로 놓친 문제

처음 테스트는 이렇게 생겼다.

```dart
expect(find.text('보일러'), findsOneWidget);
await tester.tap(find.text('켜기'));
```

개발자 눈에는 통과다. 그런데 아이콘만 있는 퀵모드 버튼은 `Text` 위젯이 없어서 테스트 대상에서 빠졌고, 상태가 바뀌어도 스크린 리더가 읽을 정보가 없었다. 더 난감한 건 `IconButton`에 `tooltip`을 넣었다고 안심했는데, 커스텀 `GestureDetector`로 만든 카드에는 Semantics 노드가 아예 없었던 점이다.

| 확인 항목 | 일반 Finder | Semantics 기반 Finder |
| --- | --- | --- |
| 화면에 글자가 보이는가 | 가능 | 가능 |
| 아이콘 버튼의 의미 | 놓칠 수 있음 | 검증 가능 |
| 켜짐·꺼짐 상태 | 별도 로직 필요 | `value`로 확인 |
| 보조공학 탭 가능 여부 | 확인 어려움 | `hasAction`으로 확인 |

그림에서 봐야 할 부분은 보이는 텍스트와 접근성 트리가 항상 같지는 않다는 점이다.

## 제어 카드에 Semantics를 명시한다

테스트를 고치기 전에 UI가 의미 있는 접근성 노드를 노출하도록 만들었다. `label`에는 위치와 기기명을 넣고, `value`에는 현재 상태를 넣었다. 상태를 `hint`에만 넣으면 TalkBack과 VoiceOver에서 일관되게 읽히지 않아 `value`를 사용했다.

```dart
class BoilerCard extends StatelessWidget {
  const BoilerCard({super.key, required this.isOn, required this.onTap});

  final bool isOn;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    return Semantics(
      container: true,
      button: true,
      label: '거실 보일러',
      value: isOn ? '켜짐' : '꺼짐',
      hint: '두 번 탭하여 상태 변경',
      child: GestureDetector(
        onTap: onTap,
        child: Row(
          children: [
            const Icon(Icons.water_damage_outlined),
            Text(isOn ? '켜기' : '끄기'),
          ],
        ),
      ),
    );
  }
}
```

`button: true`만 넣는다고 탭 액션이 생기는 것은 아니다. `Semantics`의 `onTap`을 직접 넣는 구조라면 그 콜백도 연결해야 한다. 위처럼 자식 `GestureDetector`가 액션을 제공하는 경우엔 실제 플랫폼에서 합쳐지는지 확인했다. 커스텀 위젯을 감싸는 방식이 복잡하면 `Semantics`에 `onTap: onTap`을 명시하는 편이 안전하다.

## integration_test에서 접근성 트리를 검사한다

테스트에서는 `bySemanticsLabel`로 이름을 찾고, `Semantics` 노드의 값과 액션을 확인한다. 이 테스트는 화면의 색상이나 레이아웃보다 제품 의미에 집중한다.

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('거실 보일러 접근성 상태를 읽을 수 있다', (tester) async {
    await tester.pumpWidget(const TestApp(initialBoilerOn: false));
    await tester.pumpAndSettle();

    final boiler = find.bySemanticsLabel('거실 보일러');
    expect(boiler, findsOneWidget);
    expect(tester.getSemantics(boiler), matchesSemantics(
      label: '거실 보일러',
      value: '꺼짐',
      hasTapAction: true,
      isButton: true,
    ));

    await tester.tap(boiler);
    await tester.pumpAndSettle();
    expect(tester.getSemantics(boiler), matchesSemantics(
      label: '거실 보일러',
      value: '켜짐',
      hasTapAction: true,
      isButton: true,
    ));
  });
}
```

여기서 `matchesSemantics`를 모든 필드에 적용하면 사소한 Flutter 버전 차이로 테스트가 깨질 수 있다. 우리 앱에서는 `label`, `value`, `hasTapAction`, `isButton`처럼 제품 계약에 해당하는 값만 고정했다. `hint`나 텍스트 방향 같은 값은 별도 접근성 스냅샷이 필요한 경우에만 추가한다.

## 실제 실행에서 확인할 것

접근성 테스트도 Fake 기기 서비스로 실행할 수 있다. BLE나 MQTT 연결을 기다리게 하면 Semantics 검증이 통신 지연에 끌려간다. `FakeBoilerRepository`가 초기 상태를 즉시 반환하게 만들고, 상태 변경 이벤트만 흉내 내는 방식이 안정적이었다.

실기기에서 한 번 더 확인할 때는 Android TalkBack과 iOS VoiceOver를 각각 켰다. Android는 `label + value`가 한 문장으로 읽혔지만, iOS에서는 `button` 역할이 중복 선언된 카드가 “버튼, 버튼”처럼 읽혔다. 컨테이너와 자식 `IconButton` 양쪽에 Semantics가 생긴 경우라서 `ExcludeSemantics`로 아이콘 자식의 중복 노드를 제거했다.

체크 기준은 세 가지로 줄였다.

- 아이콘만 있는 제어도 위치와 기기명이 읽힌다.
- 명령 전후의 상태가 `value`에서 바뀐다.
- 비활성 상태에서는 탭 액션이 노출되지 않는다.

## 짧게 정리하면

Flutter integration_test의 `find.text`는 화면 검증에는 편하지만 접근성 품질을 보장하지 않는다. IoT 앱의 제어 컴포넌트에는 `Semantics`로 label과 value를 명시하고, 테스트에서는 실제 사용자가 듣게 될 이름·상태·액션을 검사하는 편이 낫다. 특히 `GestureDetector` 기반 커스텀 카드와 중첩된 `IconButton`은 Semantics 트리를 직접 확인해야 한다.
