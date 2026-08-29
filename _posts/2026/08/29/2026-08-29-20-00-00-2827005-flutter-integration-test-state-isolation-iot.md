---
layout: post
title: "Flutter integration_test 테스트 격리 - Realm·GetX 상태 오염 해결"
description: "Flutter integration_test에서 Realm, shared_preferences, GetX 싱글턴이 테스트 사이에 남는 상태 오염을 재현하고, 앱 재시작과 초기화 훅으로 IoT E2E 테스트를 격리하는 방법을 정리했다."
date: 2026-08-29
tags: [Flutter, Dart, integration_test, Realm, GetX, IoT, 테스트]
comments: true
share: true
---

![Flutter integration_test 테스트 격리와 IoT 앱 상태 초기화](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 테스트가 끝나도 남아 있는 로컬 DB와 싱글턴 상태다. 화면만 새로 그린다고 이전 테스트의 기기와 MQTT 상태가 사라지지는 않는다.

Flutter integration_test는 테스트 파일을 실행할 때 앱 프로세스를 계속 유지한다. 그래서 `testWidgets`마다 새 위젯 트리가 만들어지는 것과 달리, Realm 인스턴스·`SharedPreferences`·GetX 의존성은 그대로 남을 수 있다. 처음엔 각 테스트의 시작 화면만 확인하면 격리된 줄 알았는데, 두 번째 테스트에서 앞 테스트의 보일러가 선택되어 실패했다.

## 어떤 상태가 오염되는가

| 상태 | 남는 이유 | 증상 |
| --- | --- | --- |
| Realm | 같은 파일을 계속 열어 둠 | 이전 기기·알림이 조회됨 |
| shared_preferences | 실제 디바이스 저장소 사용 | 로그인 토큰과 마지막 공간이 남음 |
| GetX | 전역 컨테이너가 유지됨 | 이전 Controller의 Stream이 계속 실행됨 |
| MQTT Fake | 연결·구독 목록을 dispose하지 않음 | 메시지가 두 번 들어옴 |

핵심은 `pumpAndSettle()`이 아니다. 이 메서드는 프레임을 안정화할 뿐, 네이티브 저장소와 전역 의존성을 초기화하지 않는다.

## 테스트 전용 초기화 훅 만들기

앱 시작 전에 테스트 모드임을 넘기고, 저장소 파일과 의존성을 테스트별로 분리한다. 아래처럼 `resetTestState()`를 앱 코드의 production 초기화와 분리해 두면 실수로 운영 데이터를 지우는 일도 피할 수 있다.

```dart
Future<void> resetTestState() async {
  await mqttService.disconnect();
  await mqttService.clearSubscriptions();

  Get.reset();
  await realm.close();

  final prefs = await SharedPreferences.getInstance();
  await prefs.clear();
}

Future<void> startTestApp(WidgetTester tester) async {
  await resetTestState();
  await tester.pumpWidget(
    MyApp(
      realmPath: 'test_${DateTime.now().microsecondsSinceEpoch}.realm',
      useFakeMqtt: true,
    ),
  );
  await tester.pumpAndSettle(const Duration(milliseconds: 300));
}
```

`Get.reset()`만 호출하면 충분하다고 생각하기 쉽지만, Realm에 저장된 사용자와 Fake MQTT의 구독은 남는다. 세 레이어를 모두 초기화해야 테스트 순서를 바꿔도 같은 결과가 나온다.

## 앱 재시작까지 검증하기

상태가 메모리에서만 사라진 건지, 실제 영속화가 정상인지 구분하려면 같은 테스트 안에서 앱을 재구성한다. 로그인 후 공간을 선택하고, 앱 루트만 다시 올린 뒤 선택 상태를 검증하는 식이다.

```dart
testWidgets('앱 재시작 뒤 선택 공간을 복원한다', (tester) async {
  await startTestApp(tester);

  await tester.tap(find.text('거실 보일러'));
  await tester.tap(find.text('선택 저장'));
  await tester.pumpAndSettle();

  await tester.pumpWidget(const SizedBox.shrink());
  await tester.pumpAndSettle();
  await startTestApp(tester);

  expect(find.text('거실 보일러'), findsOneWidget);
});
```

여기서 테스트마다 랜덤 Realm 경로를 쓰면 이전 실행의 파일을 잘못 읽을 가능성이 줄어든다. 다만 파일 삭제를 생략하면 CI 디스크가 쌓일 수 있으니 `tearDownAll`에서 테스트 디렉터리를 정리해야 한다. 실제 기기에서는 앱 데이터 삭제 권한과 플랫폼별 저장소 캐시가 달라, 최소 한 번은 Android와 iOS에서 각각 확인하는 편이 안전하다.

## 짧게 정리하면

- `pumpAndSettle()`은 상태 초기화 도구가 아니다.
- Realm, shared_preferences, GetX, MQTT Fake를 각각 reset·dispose한다.
- 테스트별 DB 경로를 분리하고 앱 재구성 시나리오를 넣는다.
- 테스트 순서를 섞어도 통과하는지 CI에서 확인한다.

해보니 테스트 격리는 테스트 코드 한 줄의 문제가 아니었다. 저장소 수명과 싱글턴 수명을 앱 실행 수명과 맞추는 설계 문제였다.
