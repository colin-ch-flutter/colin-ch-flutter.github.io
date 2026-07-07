---
layout: post
title: "Flutter IoT Riverpod Widget Test - ProviderScope overrides로 BLE 화면 고정하기"
description: "Flutter IoT 앱에서 Riverpod ProviderScope overrides로 Flutter BLE와 MQTT 상태를 고정해 Widget Test가 실제 기기 없이 흔들리지 않게 만든 방법이다."
date: 2026-07-07
tags: [Flutter, Riverpod, BLE, MQTT, IoT]
comments: true
share: true
---

![Flutter IoT Riverpod Widget Test](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

이 그림에서 봐야 할 건 실제 기기 없이도 화면 상태를 재현할 수 있는 테스트 환경이다.

Flutter IoT 앱에서 Riverpod Widget Test를 할 때는 `ProviderScope(overrides:)`로 Flutter BLE와 MQTT 상태를 고정하는 편이 제일 덜 흔들렸다. Flutter 스마트홈 화면은 실제 기기, 권한 다이얼로그, MQTT 세션에 너무 쉽게 끌려간다. 처음엔 Fake Repository만 만들면 끝이라고 생각했는데, `pumpWidget()` 안에서 앱 루트와 같은 ProviderScope를 안 쓰면 화면이 운영 Provider를 그대로 읽어버렸다.

문제는 테스트가 느린 게 아니었다. 실패 이유가 매번 달랐다. BLE 스캔 결과가 비어 있거나, MQTT 연결 상태가 `connecting`에서 멈추거나, 캐시 복원보다 카드 렌더링이 앞섰다. 실제 구현은 맞는데 테스트가 통신 타이밍을 기다리느라 흔들리는 상황이었다.

| 테스트 대상 | 고정할 Provider | 이유 |
|---|---|---|
| BLE 등록 화면 | `bleScanProvider` | 주변 기기 목록을 결정한다 |
| 보일러 카드 | `deviceCardProvider(deviceId)` | 온도, 전원, 연결 상태를 한 번에 검증한다 |
| MQTT 에러 배너 | `mqttSessionProvider` | 연결 끊김 문구를 재현한다 |
| 인증 필요 화면 | `authStateProvider` | 로그인 리다이렉트와 분리한다 |

핵심은 Widget Test에서 앱 전체를 띄우지 않는 것이다. 화면 하나를 테스트하되, 그 화면이 읽는 Provider만 명시적으로 갈아끼운다. 아래 코드는 BLE 등록 화면을 실제 `flutter_blue_plus` 없이 고정하는 형태다.

```dart
testWidgets('BLE 기기 목록을 보여준다', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        bleScanProvider.overrideWith(
          (ref) => Stream.value([
            const BleDeviceSummary(
              id: 'AA:BB:CC:DD:EE:FF',
              name: 'Living Room Boiler',
              rssi: -54,
            ),
          ]),
        ),
      ],
      child: const MaterialApp(
        home: BleRegisterPage(),
      ),
    ),
  );

  await tester.pump();

  expect(find.text('Living Room Boiler'), findsOneWidget);
  expect(find.text('-54 dBm'), findsOneWidget);
});
```

여기서 헷갈렸던 건 `overrideWithValue`와 `overrideWith` 선택이었다. 이미 만들어진 Fake 객체를 넣을 때는 `overrideWithValue`가 편하다. 반대로 Provider 안에서 `ref`를 써야 하거나 테스트마다 다른 Stream을 만들면 `overrideWith`가 낫다. 나는 처음에 전부 `overrideWithValue`로 밀어붙이다가 StreamController를 테스트 밖에서 닫아야 해서 정리가 더 복잡해졌다.

상태별 테스트는 화면 문구를 기준으로 나누는 편이 좋았다.

```dart
testWidgets('MQTT 연결 끊김 배너를 보여준다', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        deviceCardProvider('boiler-1').overrideWithValue(
          const DeviceCardView(
            title: '거실 보일러',
            powerOn: false,
            temperature: 22,
            online: false,
            message: 'MQTT 연결 끊김',
          ),
        ),
      ],
      child: const MaterialApp(
        home: DeviceCard(deviceId: 'boiler-1'),
      ),
    ),
  );

  expect(find.text('MQTT 연결 끊김'), findsOneWidget);
});
```

이 방식의 장점은 테스트가 UI 계약을 드러낸다는 점이다. Repository가 어떤 API를 호출했는지보다, 사용자가 어떤 상태를 보는지가 테스트의 중심이 된다. 보일러 전원이 꺼졌는지, 오프라인 배너가 보이는지, BLE 기기명이 노출되는지를 빠르게 확인할 수 있다.

주의할 점도 있다. `ProviderScope`를 중첩해서 앱 루트 전체를 복사하면 테스트가 다시 무거워진다. 필요한 화면과 Provider만 감싸야 한다. 그리고 Fake Stream이 끝나지 않으면 `pumpAndSettle()`이 멈출 수 있다. 내가 확인한 기준으론 실시간 스트림 화면은 `pumpAndSettle()`보다 `pump()`를 1~2번 명시적으로 호출하는 쪽이 실패 원인을 찾기 쉬웠다.

짧게 정리하면 이렇다. Flutter IoT Widget Test에서 Riverpod overrides는 실제 BLE 기기와 MQTT 서버를 떼어내는 테스트 경계다. 단위 테스트는 ProviderContainer로 상태 전이를 보고, 화면 테스트는 ProviderScope overrides로 표시 결과를 고정한다. 이 선을 나누면 테스트가 빠르고, 실패했을 때도 통신 문제인지 UI 문제인지 덜 헷갈린다.
