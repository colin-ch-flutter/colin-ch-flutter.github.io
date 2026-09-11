---
layout: post
title: "Flutter WidgetTester AlertDialog 테스트 - MQTT 명령 확인과 barrierDismissible 검증"
description: "Flutter WidgetTester로 스마트홈 MQTT 명령 전 AlertDialog의 확인·취소 흐름과 barrierDismissible 동작을 검증하는 방법을 실제 실패 사례와 함께 정리했다."
date: 2026-09-11
tags: [Flutter, Dart, 테스트, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester로 스마트홈 제어 화면의 사용자 상호작용을 검증하는 테스트 흐름](/assets/images/flutter-widgettester-scrollable-list-iot.png)

이 그림에서 볼 부분은 화면을 직접 누르는 사용자 흐름을 테스트 코드로 재현하고, 그 결과를 상태 변화까지 확인하는 방식이다.

Flutter WidgetTester에서 `AlertDialog`는 보이는 문구만 찾으면 끝날 것 같지만, MQTT 명령을 보내는 스마트홈 화면에서는 확인·취소와 바깥 영역 탭을 각각 검증해야 한다. 처음엔 다이얼로그가 떴다는 것만 검사했는데, 취소를 눌러도 `publish()`가 호출되는 버그를 놓쳤다. 실제 기기에서 보일 때까지 발견하지 못한 이유도 테스트가 대화상자의 결과를 확인하지 않았기 때문이다.

## 확인 버튼과 취소 버튼을 분리한다

테스트 대상은 보일러 전원 버튼이다. 확인을 눌렀을 때만 Repository가 MQTT 명령을 보내고, 취소하면 호출 횟수가 0이어야 한다. 테스트 더블은 `FakeMqttRepository`처럼 호출 횟수와 마지막 토픽만 기록하면 충분하다.

```dart
testWidgets('확인할 때만 보일러 명령을 발행한다', (tester) async {
  final repository = FakeMqttRepository();

  await tester.pumpWidget(
    MaterialApp(home: BoilerPage(repository: repository)),
  );

  await tester.tap(find.byKey(const Key('boiler-power')));
  await tester.pumpAndSettle();
  expect(find.text('보일러 전원을 켤까요?'), findsOneWidget);

  await tester.tap(find.text('취소'));
  await tester.pumpAndSettle();
  expect(repository.publishCount, 0);

  await tester.tap(find.byKey(const Key('boiler-power')));
  await tester.pumpAndSettle();
  await tester.tap(find.text('확인'));
  await tester.pumpAndSettle();

  expect(repository.publishCount, 1);
  expect(repository.lastTopic, 'home/boiler/set');
});
```

여기서 `pumpAndSettle()`은 다이얼로그가 열리고 닫히는 애니메이션이 끝난 뒤 Finder를 검사하기 위해 넣었다. `pump()` 한 번만 호출하면 프레임이 아직 정리되지 않아 간헐적으로 `findsNothing`이 나왔다.

## barrierDismissible은 정책을 명시한다

실수로 화면 바깥을 눌렀을 때 명령이 실행되면 곤란하다. 확인이 필요한 명령은 `showDialog`의 `barrierDismissible: false`를 명시하고, 바깥 탭 뒤에도 다이얼로그가 남아 있는지 검사한다.

| 상황 | 기대 결과 | 명령 발행 |
|---|---|---:|
| 확인 탭 | 다이얼로그 닫힘 | 1회 |
| 취소 탭 | 다이얼로그 닫힘 | 0회 |
| 바깥 영역 탭 | 다이얼로그 유지 | 0회 |

```dart
await tester.tapAt(const Offset(5, 5));
await tester.pumpAndSettle();

expect(find.text('보일러 전원을 켤까요?'), findsOneWidget);
expect(repository.publishCount, 0);
```

다만 모든 대화상자에 `false`를 적용하면 닫기 동작이 답답해진다. 로그아웃 안내처럼 되돌릴 수 있는 작업은 바깥 탭을 허용하고, 보일러·도어락처럼 즉시 장치 상태가 바뀌는 작업만 차단하는 식으로 나눴다.

## 테스트에서 헷갈렸던 지점

`find.text('확인')`이 화면 아래 다른 버튼까지 잡는 경우가 있었다. 버튼에 `Key`를 붙이거나 `find.descendant`로 다이얼로그 내부 범위를 좁히면 해결된다. 또 다이얼로그가 닫힌 뒤 SnackBar가 표시된다면 `pumpAndSettle()`이 지나치게 오래 기다릴 수 있으니, 애니메이션 정책을 확인하고 필요한 프레임만 `pump`하는 편이 낫다.

정리하면 `AlertDialog` 테스트는 표시 여부보다 결과를 검증해야 한다. 확인·취소·바깥 탭을 분리하고, MQTT Repository 호출 횟수까지 확인하면 사용자 실수로 명령이 발행되는 문제를 위젯 테스트 단계에서 잡을 수 있다.
