---
layout: post
title: "Flutter integration_test BLE 스캔 이벤트 순서 검증 - 중복 발견과 종료 타이밍 잡기"
description: "Flutter integration_test에서 실제 BLE 기기 없이 Fake 스트림으로 발견·중복·스캔 종료 이벤트 순서를 검증하고 flaky 테스트를 줄이는 방법을 정리했다."
date: 2026-08-17
tags: [Flutter, integration_test, BLE, flutter_blue_plus, IoT, 스마트홈]
comments: true
share: true
---

![Flutter BLE 스캔 이벤트 테스트 흐름](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

Flutter integration_test에서 BLE 연결 성공만 확인하면 충분한 줄 알았다. 해보니 더 자주 깨지는 곳은 스캔 결과가 들어오는 **순서**였다. 같은 기기가 두 번 발견되고, `stopScan()` 직전에 늦은 이벤트가 들어오고, 화면은 이미 닫혔는데 결과 Stream이 계속 살아 있었다. 실제 BLE 기기 없이도 이 흐름을 Fake 스트림으로 고정하면 테스트가 훨씬 덜 흔들린다.

## BLE 스캔 테스트가 자주 틀리는 이유

`flutter_blue_plus`의 스캔 결과는 한 번에 완성된 목록으로 오지 않는다. 광고 패킷이 들어올 때마다 결과가 갱신되고, 같은 기기의 RSSI가 바뀌면 또 이벤트가 발생한다.

| 상황 | 화면에서 기대하는 동작 | 테스트해야 할 것 |
|---|---|---|
| 첫 발견 | 기기 카드 추가 | `deviceId` 저장 |
| 같은 기기 재발견 | 카드 하나만 유지 | 중복 제거 |
| 스캔 종료 직전 이벤트 | 결과 반영 후 종료 | 이벤트 순서 |
| 화면 이탈 | 구독 정리 | 늦은 이벤트 무시 |

처음에는 `pumpAndSettle()`만 호출하면 된다고 생각했다. 하지만 Stream 이벤트와 타이머가 섞이면 끝나는 시점이 달라졌다. Fake에 이벤트를 직접 넣고 단계마다 화면 상태를 확인하는 편이 정확했다.

## Fake BLE 스트림 만들기

코드 바로 위에 Fake를 두는 이유는 실제 스캔을 흉내 내면서도 이벤트 발생 시점을 테스트가 소유하게 만들기 위해서다.

```dart
class FakeBleScanner {
  final _controller = StreamController<BleScanResult>.broadcast();
  bool stopped = false;

  Stream<BleScanResult> get results => _controller.stream;

  void emit(String id, int rssi) {
    if (!stopped) {
      _controller.add(BleScanResult(id: id, rssi: rssi));
    }
  }

  Future<void> stopScan() async {
    stopped = true;
  }

  Future<void> dispose() => _controller.close();
}
```

실제 앱에서는 `BleScanner` 인터페이스를 만들고 `flutter_blue_plus` 구현체와 이 Fake를 주입한다. `stopped` 검사는 화면이 닫힌 뒤 늦은 광고 이벤트를 무시하는지 확인하기 위해 필요하다.

## integration_test에서 순서 고정하기

테스트에서는 이벤트 하나를 넣을 때마다 프레임을 진행한다.

```dart
testWidgets('BLE 스캔은 같은 기기를 한 번만 표시한다', (tester) async {
  final scanner = FakeBleScanner();
  await tester.pumpWidget(TestApp(scanner: scanner));

  scanner.emit('boiler-01', -48);
  await tester.pump();
  expect(find.text('boiler-01'), findsOneWidget);

  scanner.emit('boiler-01', -42); // RSSI만 변경
  await tester.pump();
  expect(find.text('boiler-01'), findsOneWidget);

  scanner.emit('boiler-02', -61);
  await tester.pump();
  expect(find.byType(BleDeviceCard), findsNWidgets(2));

  await scanner.stopScan();
  scanner.emit('boiler-03', -70); // 종료 후 이벤트는 무시
  await tester.pump();
  expect(find.byType(BleDeviceCard), findsNWidgets(2));

  await scanner.dispose();
});
```

핵심은 `pumpAndSettle()` 대신 이벤트 단위로 `pump()`하는 것이다. 실제 프로젝트에서는 화면뿐 아니라 Repository의 `scanStopCalls`도 검증했다. 화면이 같아도 `stopScan()`이 두 번 호출되는 버그는 남을 수 있다.

## 주의할 점

Fake가 처음부터 `stopped = true`면 운영 흐름을 검증하지 못한다. `발견 → RSSI 갱신 → 다른 기기 발견 → 종료` 네 단계를 넣는다. 테스트가 끝날 때 StreamController도 닫아야 이전 구독이 남지 않는다.

솔직하게 정리하면 BLE integration_test의 핵심은 실제 기기를 연결하는 데 있지 않다. 실기기 연결은 별도 테스트로 확인하고, CI에서는 이벤트 순서와 정리 책임을 Fake로 고정하는 편이 재현성이 높다.

짧게 요점만 남기면:

- BLE 결과는 목록이 아니라 시간 순서가 있는 Stream 이벤트다.
- 같은 `deviceId`의 재발견은 카드 수가 늘지 않는지 검증한다.
- `stopScan()` 이후 늦은 이벤트와 Stream dispose를 반드시 테스트한다.
- `pumpAndSettle()`보다 이벤트별 `pump()`이 원인 파악에 유리하다.
