---
layout: post
title: "Flutter integration_test 성능 회귀 테스트 - IoT 화면 프레임 드롭과 메모리 증가 잡기"
description: "Flutter integration_test로 IoT 스마트홈 화면의 프레임 드롭과 메모리 증가를 반복 측정하고, MQTT·BLE 시나리오에서 성능 회귀를 CI 기준선으로 잡는 방법을 정리했다."
date: 2026-08-20
tags: [Flutter, Dart, IoT, MQTT, BLE, CI/CD]
comments: true
share: true
---

![Flutter integration_test 성능 회귀 테스트와 IoT 화면 측정](/assets/images/flutter-integration-test-parallel-ci-iot.png)

이 그림에서 봐야 할 점은 기능 성공 여부와 성능 기준선을 같은 테스트 흐름에서 함께 기록한다는 것이다.

Flutter integration_test에서 버튼 클릭이 통과하는 것만으로는 IoT 화면이 안정적이라고 말하기 어렵다. MQTT 상태가 계속 들어오고 BLE 기기 목록이 갱신되는 화면은 10분쯤 사용했을 때 프레임이 밀리거나 메모리가 조금씩 늘어날 수 있다. 이번에는 기능 테스트에 성능 기준선을 붙여서, 눈에 보이는 문제가 되기 전에 잡는 흐름을 만들었다.

## 처음엔 프로파일러만 보면 되는 줄 알았다

처음엔 DevTools Memory 화면을 열고 수동으로 스크롤하면 충분하다고 생각했다. 그런데 같은 시나리오를 세 번 실행할 때마다 결과가 달랐다. BLE 스캔이 끝나기 전에 MQTT 메시지가 몰리면 프레임 시간이 튀었고, 수동 측정이라 CI에서는 아무것도 남지 않았다.

그래서 테스트가 직접 측정할 값은 두 가지로 줄였다.

| 측정값 | 기준선 | 실패로 보는 경우 |
|---|---:|---|
| 90프레임 중 느린 프레임 | 16ms 초과 8개 이하 | 9개 이상 |
| 시나리오 전후 메모리 | 증가 12MB 이하 | 12MB 초과 |

## 테스트에서 프레임 시간을 수집하기

`FrameTiming`은 화면에 실제 프레임이 그려진 뒤 확인해야 하므로, 버튼을 누른 직후가 아니라 `pumpAndSettle()` 뒤에 샘플을 수집한다. 앱 코드에 테스트 전용 측정기를 주입하면 테스트가 Flutter binding에 과하게 묶이지 않는다.

```dart
class PerformanceProbe {
  final List<FrameTiming> timings = [];

  void start() {
    WidgetsBinding.instance.addTimingsCallback(timings.addAll);
  }

  int get slowFrameCount => timings
      .where((t) => t.totalSpan > const Duration(milliseconds: 16))
      .length;

  void dispose() {
    WidgetsBinding.instance.removeTimingsCallback(timings.addAll);
  }
}

testWidgets('기기 목록 갱신 뒤 프레임 기준선을 넘지 않는다', (tester) async {
  final probe = PerformanceProbe()..start();
  addTearDown(probe.dispose);

  await tester.pumpWidget(const TestApp());
  await tester.tap(find.text('기기 새로고침'));
  await tester.pumpAndSettle(const Duration(milliseconds: 500));

  expect(probe.slowFrameCount, lessThanOrEqualTo(8));
});
```

실제 화면에서는 FakeMqttService가 100ms 간격으로 상태를 보내고, FakeBleService가 기기 20개를 순서대로 추가하도록 만들었다. 실서버를 붙이면 네트워크 지연이 섞여서 UI 성능을 제대로 비교할 수 없다.

## 메모리 증가는 전후 차이만 본다

Dart VM의 메모리 숫자는 실행마다 조금씩 달라진다. 절대값을 `expect`하면 GC 시점 때문에 flaky 테스트가 된다. 대신 동일한 화면 진입·퇴장 시나리오를 수행하고, 측정 전후의 증가량만 제한했다.

```dart
final before = await memoryUsageInBytes();

for (var i = 0; i < 5; i++) {
  await tester.tap(find.text('스마트홈'));
  await tester.pumpAndSettle();
  await tester.tap(find.text('뒤로'));
  await tester.pumpAndSettle();
}

final after = await memoryUsageInBytes();
expect(after - before, lessThan(12 * 1024 * 1024));
```

여기서 `memoryUsageInBytes()`는 테스트 앱에서 `ServiceExtension`으로 VM 메모리를 읽는 래퍼다. 중요한 건 구현보다 같은 조건을 반복하는 것이다. 첫 실행의 캐시 생성 비용까지 누수로 오해하지 않도록 워밍업 한 번을 버리고, 측정 시나리오를 그 뒤에 실행했다.

## CI에서 성능 테스트를 다루는 기준

성능 테스트는 모든 PR에서 돌리면 느리고, 매번 실패하면 개발자가 경고를 무시하게 된다. 그래서 GitHub Actions에서는 일반 smoke 테스트와 분리해 하루 한 번, 그리고 `performance` 라벨이 붙은 변경에서만 실행했다.

| 실행 환경 | 목적 | 판정 |
|---|---|---|
| 로컬 에뮬레이터 | 빠른 회귀 확인 | 참고용 로그 |
| 고정 Android 실기기 | 프레임·메모리 기준선 | PR 차단 |
| iOS 실기기 | 플랫폼별 차이 확인 | 야간 리포트 |

솔직하게 정리하면 성능 기준선 숫자는 모든 기기에 그대로 적용할 수 없다. 기기 모델과 화면 해상도가 바뀌면 다시 기준을 잡아야 한다. 그래도 “느린 것 같다”는 감상 대신, 같은 IoT 시나리오에서 프레임과 메모리가 얼마나 변했는지를 남기는 것만으로 디버깅 시간이 크게 줄었다.
