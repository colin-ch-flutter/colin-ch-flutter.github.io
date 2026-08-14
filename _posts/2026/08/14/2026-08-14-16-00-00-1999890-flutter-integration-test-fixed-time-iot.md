---
layout: post
title: "Flutter integration_test 시간 고정 테스트 - IoT 예약 제어의 DateTime.now() 문제 해결"
description: "Flutter integration_test에서 DateTime.now() 때문에 자정과 타임존 경계에서 깨지는 IoT 예약 제어를 package:clock과 테스트용 시간 주입으로 고정하는 방법을 정리했다."
date: 2026-08-14
tags: [Flutter, Dart, integration_test, IoT, 스마트홈, CI/CD]
comments: true
share: true
---
![Flutter integration_test IoT 예약 제어 시간 고정 테스트](/assets/images/flutter-integration-test-ci-report-iot.png)
이 그림에서 볼 부분은 테스트가 실행된 실제 시각이 아니라, 시나리오가 정한 고정 시각을 기준으로 결과를 판단한다는 점이다.

Flutter integration_test에서 IoT 예약 제어를 검증할 때 `DateTime.now()`를 그대로 쓰면 자정, 타임존, 서머타임 경계에서 테스트가 흔들린다. 내 경우에는 오전 7시 보일러 예약을 확인하는 테스트가 로컬에서는 통과했지만 CI에서는 예약 카드가 “대기 중”으로 남았다. 실행 시간이 한국 시간 자정 근처였고, 앱과 테스트가 서로 다른 타임존을 보고 있었다.

## 실제 시각을 읽는 코드가 만드는 문제

예약 기능은 현재 시각과 예약 시각을 비교한다. 그런데 앱 내부에서 `DateTime.now()`를 직접 호출하면 integration_test가 원하는 시각을 만들 수 없다. 테스트 시작 시각을 바꿔도 앱 프로세스가 읽는 시계는 바뀌지 않는다.

| 시나리오 | 실제 시계에 의존할 때 | 고정 시계를 넣었을 때 |
|---|---|---|
| 예약 1분 전 | 실행 시각에 따라 결과가 달라짐 | 항상 `예약 대기` |
| 예약 시각 도달 | 자정 경계에서 간헐 실패 | 항상 `자동 켜짐` |
| 만료된 예약 | CI 타임존에 따라 판정 변경 | UTC 기준으로 재현 가능 |

핵심은 테스트 코드에서 시간을 바꾸는 것이 아니라, 앱이 시간을 읽는 통로를 하나로 만드는 것이다. Repository와 Controller가 `Clock`을 전달받으면 integration_test에서는 같은 시계를 주입할 수 있다.

## Clock을 앱 진입점까지 전달하기

`package:clock`을 사용하면 운영 코드의 시간 읽기를 감싸고, 테스트에서는 원하는 시각을 반환하게 만들 수 있다. 실제 앱에서는 기본 `Clock()`을 쓰고 테스트 앱에서는 `Clock.fixed()`를 넘긴다.

```dart
import 'package:clock/clock.dart';

class ScheduleRepository {
  ScheduleRepository({Clock? clock}) : _clock = clock ?? const Clock();

  final Clock _clock;

  bool shouldStart(DateTime scheduledAt) {
    final now = _clock.now().toUtc();
    return !now.isBefore(scheduledAt.toUtc());
  }
}

Future<void> main() async {
  final repository = ScheduleRepository();
  runApp(SmartHomeApp(scheduleRepository: repository));
}
```

테스트 앱을 띄울 때만 고정 시각을 주입한다. `2026-08-14 06:59:59`와 `07:00:00`을 분리하면 경계값도 명확해진다.

```dart
final testNow = DateTime.utc(2026, 8, 14, 7);
final repository = ScheduleRepository(clock: Clock.fixed(testNow));

await integrationTestApp(
  scheduleRepository: repository,
  schedule: DateTime.utc(2026, 8, 14, 7),
);

expect(find.text('자동 켜짐'), findsOneWidget);
expect(find.text('예약 대기'), findsNothing);
```

테스트 전용 `integrationTestApp`은 실제 로그인과 MQTT 연결을 생략하고 Fake Repository를 주입하는 헬퍼다. 이 구조가 없으면 테스트마다 앱 초기화 코드를 복사하게 되고, 어떤 테스트는 고정 시계를 쓰고 어떤 테스트는 실제 시계를 쓰는 식으로 다시 섞인다.

## 타임존을 섞지 않는 기준

처음에는 예약 시간을 `DateTime(2026, 8, 14, 7)`로 만들고 비교만 `toUtc()`로 처리했다. 이 방식은 개발자 컴퓨터의 로컬 타임존에 따라 값이 달라졌다. 저장·비교·테스트 입력을 모두 UTC로 통일하고 화면에 표시할 때만 로컬 시간으로 바꾸니 문제가 사라졌다.

```dart
testWidgets('예약 시각이 되면 보일러를 켠다', (tester) async {
  final clock = Clock.fixed(DateTime.utc(2026, 8, 14, 7));
  final repo = ScheduleRepository(clock: clock);

  await pumpTestApp(
    tester,
    repository: repo,
    scheduledAt: DateTime.utc(2026, 8, 14, 7),
  );

  await tester.pumpAndSettle();
  expect(find.text('보일러 가동 중'), findsOneWidget);
});
```

여기서 `pumpAndSettle()`은 시계를 흐르게 하지 않는다. 예약 판정이 Timer에 묶여 있다면 시간 주입만으로 끝나지 않고, Timer를 수동으로 진행하는 별도 단위 테스트가 필요하다. integration_test에서는 “앱을 켰을 때 이미 예약 시각이 지났는가”처럼 화면과 상태 연결을 검증하고, 분 단위 타이머 자체는 `fakeAsync`로 분리하는 편이 안정적이다.

## 정리

Flutter integration_test의 시간 관련 flaky는 대기 시간을 늘려 해결할 문제가 아니었다. `DateTime.now()`를 앱 곳곳에서 호출하지 않게 만들고, 예약 데이터는 UTC로 저장하며, 테스트 앱에는 고정 `Clock`을 주입해야 한다. 특히 자정 경계와 CI 타임존을 한 번씩 의도적으로 테스트하면 “내 컴퓨터에서는 통과”하는 IoT 예약 버그를 빨리 잡을 수 있다.
