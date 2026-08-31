---
layout: post
title: "Flutter integration_test 성능 검증 - IoT 대시보드 프레임 드랍과 rebuild 잡기"
description: "Flutter integration_test에서 IoT 대시보드의 프레임 드랍과 과도한 rebuild를 측정하는 방법을 정리했다. FrameTiming과 rebuild 카운터로 기준을 만들고 flaky 테스트를 피하는 구조까지 다룬다."
date: 2026-08-31
tags: [Flutter, Dart, IoT, 스마트홈, 테스트, 성능최적화, CI/CD]
comments: true
share: true
---

![Flutter integration_test로 IoT 대시보드 프레임 드랍과 rebuild 성능 검증](/assets/images/flutter-integration-test-performance-frame-drop-iot.png)

Flutter integration_test에서 버튼이 눌리고 MQTT 응답이 왔다는 것만 확인하면 성능 회귀를 놓친다. IoT 대시보드는 센서 스트림이 계속 들어오고 카드가 여러 개라서, 작은 상태 변경 하나가 전체 화면 rebuild로 번지기 쉽다. 이번에는 `FrameTiming`으로 렌더링 시간을 측정하고, rebuild 횟수까지 함께 검사하는 기준을 만들었다.

## 화면은 정상인데 스크롤이 끊겼다

처음엔 센서 Fake가 너무 자주 이벤트를 보내서 생긴 문제라고 생각했다. 그런데 이벤트 간격을 1초로 늘려도 기기 목록을 스크롤할 때 간헐적으로 끊겼다. 원인은 `Obx` 범위가 카드 하나가 아니라 대시보드 전체를 감싸고 있었고, 온도 한 건이 바뀔 때마다 12개 카드가 다시 그려지는 구조였다.

성능 테스트에서 고정한 조건은 아래와 같다.

| 검사 대상 | 기준 | 실패로 보는 상황 |
| --- | --- | --- |
| 화면 전환 | 1초 안에 안정화 | 네트워크 대기로 화면이 멈춤 |
| 프레임 | 실제 수집 구간의 평균과 99퍼센타일 | 특정 프레임만 과도하게 느림 |
| rebuild | 카드별 변경 범위 안에서만 발생 | 전체 카드가 함께 rebuild |
| 스크롤 | 목록 끝까지 이동 | 프레임 드랍이 반복됨 |

기준값은 에뮬레이터 종류와 CI 머신에 따라 달라진다. 그래서 “항상 16ms 이하”처럼 단일 숫자를 박아 넣지 않고, 동일한 실행 환경에서 기준선을 먼저 기록했다.

## FrameTiming을 테스트 구간에만 모은다

`addTimingsCallback`을 테스트 파일 전체에 걸어두면 앱 시작 애니메이션과 테스트 대상 화면의 수치가 섞인다. 성능 검증 직전에 콜백을 등록하고, 스크롤이 끝난 직후 제거하는 방식이 훨씬 해석하기 쉽다.

아래 헬퍼는 UI 스레드와 raster 스레드 중 더 느린 시간을 프레임 시간으로 사용한다.

```dart
Future<List<FrameTiming>> collectTimings(
  WidgetTester tester,
  Future<void> Function() action,
) async {
  final timings = <FrameTiming>[];
  final binding = tester.binding;

  void onTimings(List<FrameTiming> value) {
    timings.addAll(value);
  }

  binding.addTimingsCallback(onTimings);
  try {
    await action();
    await tester.pumpAndSettle(const Duration(milliseconds: 200));
  } finally {
    binding.removeTimingsCallback(onTimings);
  }
  return timings;
}

double p99RasterMs(List<FrameTiming> timings) {
  if (timings.isEmpty) return 0;
  final values = timings
      .map((timing) => timing.rasterDuration.inMicroseconds / 1000)
      .toList()
    ..sort();
  final index = ((values.length - 1) * .99).round();
  return values[index];
}
```

테스트에서는 실제 스크롤 동작만 수집했다. 초기 build 비용까지 포함하면 앱을 띄우는 위치에 따라 결과가 크게 흔들리기 때문이다.

```dart
testWidgets('기기 목록 스크롤의 p99 프레임 시간이 기준 안에 든다',
    (tester) async {
  await tester.pumpWidget(
    TestApp(repository: FakeBoilerRepository(count: 30)),
  );
  await tester.pumpAndSettle();

  final timings = await collectTimings(tester, () async {
    await tester.fling(
      find.byKey(const Key('device-list')),
      const Offset(0, -900),
      1800,
    );
  });

  expect(timings, isNotEmpty);
  expect(p99RasterMs(timings), lessThan(34));
});
```

여기서 `34ms`는 60fps의 16.67ms보다 느슨한 값이다. 테스트 환경의 편차를 감안한 경고선이며, 제품 목표가 60fps라면 성능 프로파일링에서 실제 병목을 따로 확인해야 한다. 이 테스트 하나로 앱의 체감 성능을 증명할 수는 없다.

## rebuild 횟수는 화면에 노출하지 않는다

rebuild 카운터를 `Text`로 화면에 표시하면 그 카운터 변경 자체가 다시 rebuild를 일으킨다. 테스트 전용 콜백으로 숫자만 수집하는 편이 안전하다.

```dart
class RebuildProbe extends StatelessWidget {
  const RebuildProbe({required this.child, required this.onBuild, super.key});

  final Widget child;
  final VoidCallback onBuild;

  @override
  Widget build(BuildContext context) {
    onBuild();
    return child;
  }
}

testWidgets('온도 변경은 해당 카드만 다시 그린다', (tester) async {
  var dashboardBuilds = 0;
  var boilerCardBuilds = 0;

  await tester.pumpWidget(TestApp(
    dashboardProbe: () => dashboardBuilds++,
    boilerCardProbe: () => boilerCardBuilds++,
  ));
  await tester.pumpAndSettle();
  final beforeDashboard = dashboardBuilds;
  final beforeCard = boilerCardBuilds;

  await tester.tap(find.byKey(const Key('fake-temperature-update')));
  await tester.pump();

  expect(dashboardBuilds - beforeDashboard, lessThanOrEqualTo(1));
  expect(boilerCardBuilds - beforeCard, greaterThanOrEqualTo(1));
});
```

처음 이 검사를 넣었을 때 `dashboardBuilds`가 8까지 올라갔다. `Obx`를 카드 단위로 좁히고, 변하지 않는 센서 아이콘 목록을 `const` 위젯으로 분리한 뒤 1~2회 수준으로 내려왔다. 숫자 자체보다 “어떤 상태 변경이 어떤 범위를 다시 그리는가”를 계약으로 남긴 것이 핵심이었다.

## CI에서 성능 테스트를 운영하는 방법

성능 테스트는 모든 PR에서 실행하면 오히려 신뢰를 잃는다. 공유 CI 러너의 CPU 상태가 매번 달라서 경계값 근처의 실패가 반복되기 때문이다. 나는 일반 integration test와 분리해 nightly job에서 같은 에뮬레이터 이미지로 실행하고, p99 값과 rebuild 횟수를 아티팩트로 저장한다.

| 실행 위치 | 목적 | 처리 |
| --- | --- | --- |
| 로컬 프로파일 모드 | 병목 위치 확인 | DevTools Timeline으로 분석 |
| PR CI | 명백한 회귀 차단 | rebuild 상한만 검사 |
| nightly 고정 환경 | 프레임 추세 확인 | p99·평균을 기록 |

주의할 점도 있다. `pumpAndSettle`은 끝나지 않는 센서 스트림이나 반복 애니메이션이 있으면 타임아웃 난다. Fake 데이터 스트림은 테스트가 끝날 때 닫고, 스크롤 직후에는 무작정 settle하지 말고 필요한 프레임 수만 `pump`하는 편이 더 안정적이다.

짧게 정리하면 `Flutter integration_test` 성능 검증은 “화면이 보인다”에서 끝나지 않는다. IoT 화면에서 실제로 자주 바뀌는 센서 상태를 주입하고, 그 구간의 `FrameTiming`과 rebuild 범위를 함께 측정해야 한다. 프레임 기준은 환경별로 분리하고, CI에서는 재현 가능한 조건과 추세 기록을 만들어야 flaky 테스트와 실제 성능 회귀를 구분할 수 있다.
