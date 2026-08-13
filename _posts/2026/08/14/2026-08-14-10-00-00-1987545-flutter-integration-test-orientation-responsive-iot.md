---
layout: post
title: "Flutter integration_test 화면 회전 테스트 - IoT 제어 화면 상태 보존과 반응형 검증"
description: "Flutter integration_test에서 세로·가로 화면 회전 뒤 IoT 제어 상태와 MQTT 연결이 유지되는지 검증하고, 반응형 레이아웃 깨짐을 잡는 테스트 패턴을 정리했다."
date: 2026-08-14
tags: [Flutter, Dart, integration_test, IoT, MQTT, Android, iOS]
comments: true
share: true
---
![Flutter integration_test 화면 회전과 IoT 반응형 레이아웃 테스트](/assets/images/flutter-integration-test-parallel-ci-iot.png)
이 그림에서 볼 부분은 기기 해상도가 달라도 테스트 대상은 같은 제어 상태라는 점이다.

Flutter integration_test에서 버튼을 눌러 보일러를 켜는 흐름만 통과하면 안 된다. 실제 사용자는 가로로 돌려 대시보드를 확인하고, 다시 세로로 돌린다. 이때 카드가 잘리거나 선택한 모드가 초기화되면 제어 앱으로서 신뢰하기 어렵다. 나는 화면 회전 테스트를 빼먹었다가 태블릿 가로 화면에서 `GridView`가 버튼을 덮는 문제를 늦게 발견했다.

## 화면 회전에서 실제로 깨지는 지점

화면 방향만 바뀌는 것처럼 보여도 앱에서는 위젯 트리가 다시 레이아웃을 계산하고, 일부 상태 객체가 재생성된다. 특히 `initState()`에서 MQTT 구독을 시작하는 구조라면 회전 때 중복 구독이 생길 수 있다.

| 확인 항목 | 실패했을 때 보이는 증상 |
|---|---|
| 제어 상태 | 켜짐 카드가 꺼짐으로 표시됨 |
| MQTT 구독 | 같은 메시지가 두 번 반영됨 |
| 레이아웃 | 가로 화면에서 버튼·차트가 잘림 |
| 입력 위치 | 회전 직후 이전 좌표를 탭함 |

상태는 화면 생명주기보다 긴 Repository에 두고, 화면은 그 상태를 구독만 하게 만든다. 테스트에서도 `FakeMqttService`가 보낸 상태가 회전 뒤 그대로 남는지 확인한다.

## 테스트 전용 회전 헬퍼

`integration_test` 자체에는 모든 플랫폼에서 동일한 화면 회전 API가 없다. 그래서 앱에 테스트 모드일 때만 노출되는 방향 전환 액션을 두고, 실제 OS 방향 변경 이벤트와 같은 레이아웃 경로를 통과시켰다.

```dart
Future<void> rotateTo(WidgetTester tester, DeviceOrientation orientation) async {
  await SystemChrome.setPreferredOrientations([orientation]);
  await tester.binding.setSurfaceSize(
    orientation == DeviceOrientation.landscapeLeft
        ? const Size(844, 390)
        : const Size(390, 844),
  );
  await tester.pumpAndSettle(const Duration(milliseconds: 500));
}

testWidgets('회전 후 보일러 상태와 MQTT 구독을 유지한다', (tester) async {
  final mqtt = FakeMqttService()..emit('boiler/1/state', {'power': true});
  await pumpTestApp(tester, mqtt: mqtt);

  expect(find.text('가동 중'), findsOneWidget);
  await rotateTo(tester, DeviceOrientation.landscapeLeft);
  expect(find.text('가동 중'), findsOneWidget);
  expect(mqtt.subscribeCount('boiler/1/state'), 1);

  await rotateTo(tester, DeviceOrientation.portraitUp);
  expect(find.text('가동 중'), findsOneWidget);
  expect(mqtt.subscribeCount('boiler/1/state'), 1);
});
```

코드에서 `setSurfaceSize`는 Widget Test 방식처럼 보이지만, 실제 디바이스 테스트에서는 플랫폼별 방향 전환 명령을 연결해야 한다. 이 헬퍼를 모든 환경에서 무조건 호출하면 CI 에뮬레이터 크기와 실제 화면 비율이 달라질 수 있으니, 레이아웃 검증과 네이티브 방향 검증을 분리했다.

## 실패를 줄인 기준

`pumpAndSettle()`만 믿으면 MQTT 스트림이 계속 살아 있는 동안 테스트가 끝나지 않을 수 있다. 회전 완료 신호, 카드가 다시 나타나는 조건, 구독 횟수를 각각 기다리는 편이 낫다. 또 Android 에뮬레이터와 iOS 실기기의 안전 영역이 달라서 `MediaQuery.padding`을 고정값으로 비교하지 않았다.

짧게 정리하면 `Flutter integration_test` 화면 회전 테스트의 핵심은 방향 자체가 아니다. 회전 후에도 Repository 상태는 유지되고, MQTT 구독은 한 번만 남으며, 가로·세로에서 제어 버튼이 실제로 눌리는지를 함께 확인하는 것이다.
