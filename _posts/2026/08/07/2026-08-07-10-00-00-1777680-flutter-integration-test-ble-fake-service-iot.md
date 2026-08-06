---
layout: post
title: "Flutter integration_test BLE 테스트 - flutter_blue_plus를 FakeBleService로 격리하기"
description: "Flutter integration_test에서 실제 BLE 기기와 flutter_blue_plus에 의존하지 않고 스캔·연결·명령 전송 흐름을 FakeBleService로 검증하는 방법을 IoT 앱 코드로 정리했다."
date: 2026-08-07
tags: [Flutter, integration_test, BLE, flutter_blue_plus, IoT, CI/CD]
comments: true
share: true
---

![Flutter integration_test BLE FakeBleService 구조](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

Flutter integration_test에서 BLE 연결을 검증할 때는 실제 기기를 테스트 데이터로 쓰면 안 된다. `flutter_blue_plus` 스캔 결과와 연결 이벤트를 FakeBleService로 바꾸면 CI에서도 같은 흐름을 빠르게 반복할 수 있다. 실제 BLE 통신은 별도 실기기 테스트로 남기고, 여기서는 “기기 발견 → 연결 → 명령 전송 → 상태 반영”을 확인하는 방식이다.

## 실제 BLE 기기로 테스트했을 때 생긴 문제

처음엔 테스트 시작 때 보일러 주변에 스마트폰을 두고 `flutter_blue_plus`를 그대로 실행했다. 해보니 테스트가 실패한 건 앱 로직이 아니라 주변 환경인 경우가 많았다.

| 상황 | 실패 원인 | 테스트에서 바꾼 기준 |
|---|---|---|
| 스캔 결과 없음 | 기기 전원·광고 주기 차이 | 고정된 FakePeripheral 반환 |
| 연결 대기 초과 | 이미 다른 앱이 연결함 | 즉시 연결되는 Fake 상태 |
| 명령 응답 지연 | BLE notify 타이밍 차이 | 명령 후 상태 이벤트 직접 발행 |
| CI 실행 불가 | 에뮬레이터에 BLE 하드웨어 없음 | 네이티브 플러그인 경계에서 격리 |

실기기 테스트 한 편은 개발자 컴퓨터에서는 통과하고 Firebase Test Lab에서는 실행조차 못 했다. 특히 iOS 시뮬레이터는 BLE 자체를 제공하지 않으므로, 화면과 Controller의 흐름을 확인하는 테스트까지 실기기에 묶어둘 이유가 없었다.

## flutter_blue_plus를 바로 호출하지 않는다

Controller가 `FlutterBluePlus.startScan()`을 직접 호출하면 `integration_test`에서 교체 지점이 사라진다. 앱 코드에서는 BLE 플러그인을 감싼 인터페이스만 알고 있도록 경계를 만든다.

아래 인터페이스가 있으면 운영 앱은 실제 구현을, 테스트 앱은 Fake 구현을 선택할 수 있다.

```dart
abstract interface class BleGateway {
  Stream<List<BleDevice>> scan();
  Future<void> connect(String deviceId);
  Future<void> write(String deviceId, List<int> command);
  Stream<BleConnectionState> connectionState(String deviceId);
}
```

실제 구현은 이 경계 안에서만 `flutter_blue_plus`를 사용한다. Controller에는 패키지 타입을 노출하지 않는 것이 핵심이다. 나중에 Android 권한 처리나 iOS 재연결 정책이 바뀌어도 화면 테스트 코드는 그대로 남는다.

## FakeBleService는 상태를 직접 재현한다

Fake의 목적은 실제 BLE를 흉내 내는 것이 아니라, 앱이 필요로 하는 상태 전이를 결정적으로 재현하는 데 있다. 그래서 타이머나 랜덤 광고 데이터를 넣지 않았다.

```dart
class FakeBleGateway implements BleGateway {
  final _state = StreamController<BleConnectionState>.broadcast();
  final devices = [
    BleDevice(id: 'boiler-001', name: 'Living Room Boiler'),
  ];

  @override
  Stream<List<BleDevice>> scan() async* {
    yield devices;
  }

  @override
  Future<void> connect(String deviceId) async {
    _state.add(BleConnectionState.connected);
  }

  @override
  Future<void> write(String deviceId, List<int> command) async {
    if (command == [0x01, 0x20]) {
      _state.add(BleConnectionState.commandCompleted);
    }
  }

  @override
  Stream<BleConnectionState> connectionState(String deviceId) => _state.stream;
}
```

명령 배열을 테스트 안에서 매번 직접 만들기보다 `BoilerCommand.heatOn` 같은 도메인 값으로 감싸는 편이 좋다. 그래야 이 테스트가 BLE 바이트 포맷을 검증하는 테스트로 변질되지 않는다.

## integration_test에서 Fake를 주입한다

테스트 전용 앱 시작점에서 Fake Gateway를 Repository에 넣는다. 기존 네트워크 격리와 같은 방식으로 외부 장치 의존성도 앱 시작 경계에서 선택한다.

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('BLE 기기 연결 후 난방 명령을 보낸다', (tester) async {
    final fakeBle = FakeBleGateway();
    await tester.pumpWidget(
      TestApp(bleGateway: fakeBle),
    );

    await tester.tap(find.text('기기 찾기'));
    await tester.pumpAndSettle();
    expect(find.text('Living Room Boiler'), findsOneWidget);

    await tester.tap(find.text('연결'));
    await tester.pump();
    expect(find.text('연결됨'), findsOneWidget);

    await tester.tap(find.text('난방 켜기'));
    await tester.pump();
    expect(find.text('난방 명령 완료'), findsOneWidget);
  });
}
```

여기서 `pumpAndSettle()`은 스캔 화면처럼 애니메이션이 끝나야 하는 구간에만 사용했다. 연결과 명령은 Fake가 즉시 상태를 발행하므로 `pump()`만 호출한다. 모든 구간에 `pumpAndSettle()`을 넣으면 실제 플러그인이 없어도 불필요하게 테스트 시간이 늘어난다.

## 실기기 테스트와 분리할 기준

Fake 테스트가 통과했다고 BLE 통신까지 정상이라는 뜻은 아니다. 다음처럼 검증 범위를 나누면 실패 원인을 빠르게 찾을 수 있다.

| 테스트 종류 | 확인할 것 | 실행 위치 |
|---|---|---|
| integration_test + Fake | 화면·Controller·Repository 흐름 | 로컬, CI |
| Android 실기기 | 권한, 스캔, GATT write | 야간 또는 수동 |
| iOS 실기기 | 백그라운드 복귀, 재연결 | 릴리스 전 |

정리하면, BLE를 완전히 가짜로 만드는 것이 아니라 플러그인 경계까지만 가짜로 만든다. 연결 상태 전이와 명령 실패 케이스를 Fake에 명시하면 CI는 빨라지고, 실제 기기에서만 확인해야 하는 문제도 분명해진다. 처음엔 실기기 하나면 충분하다고 생각했는데, 테스트가 쌓일수록 장치를 테스트 러너에서 분리한 판단이 맞았다.
