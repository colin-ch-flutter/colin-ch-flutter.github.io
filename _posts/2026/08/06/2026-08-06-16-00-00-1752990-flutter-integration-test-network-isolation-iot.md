---
layout: post
title: "Flutter integration_test 네트워크 격리 - IoT 앱을 실제 서버 없이 검증하기"
description: "Flutter integration_test에서 실제 API와 MQTT에 연결돼 테스트가 느려지고 흔들리는 문제를 AppConfig와 Fake Repository 주입으로 격리하는 방법을 IoT 앱 코드로 정리했다."
date: 2026-08-06
tags: [Flutter, integration_test, IoT, MQTT, CleanArchitecture, CI/CD]
comments: true
share: true
---

![Flutter integration_test 네트워크 격리 구조](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

Flutter integration_test가 Firebase Test Lab에서 간헐적으로 실패한다면 네트워크를 테스트 범위에서 빼야 한다. IoT 앱에서 실제 API와 MQTT 브로커까지 연결하면 테스트가 성공해도 20초가 걸리고, 서버 응답이 늦으면 타이밍 테스트처럼 변한다. 이번에는 화면과 실제 통신 계층 사이에 Fake Repository를 넣어 앱 전체 흐름은 유지하되 외부 네트워크만 끊었다.

## 실제 서버 연결이 flaky를 만든다

처음엔 `integration_test`가 앱을 실제로 실행하니 개발 서버를 붙이는 게 당연하다고 생각했다. 해보니 세 가지 문제가 생겼다.

| 의존성 | 테스트에서 생긴 문제 | 격리 기준 |
|---|---|---|
| REST API | 서버 데이터에 따라 집 목록이 달라짐 | 고정된 Fake Repository |
| MQTT | 브로커 연결·구독 대기 시간이 발생 | FakeMqttGateway |
| 현재 시간 | 스케줄 버튼 상태가 실행 시각에 따라 달라짐 | 고정 Clock |

integration_test의 목적은 서버의 가용성이 아니라 앱이 로딩 상태에서 제어 완료 상태로 이동하는지 확인하는 데 있다. 서버 장애 테스트는 별도 계약 테스트로 분리하는 편이 맞다.

## 실행 환경을 앱 시작점에서 주입한다

`main()` 안에서 Repository를 무조건 실서비스로 만들면 테스트가 시작된 뒤 교체할 방법이 없다. 앱 시작점에서 환경을 받도록 바꾸면 테스트는 Fake 구현을 선택할 수 있다.

아래처럼 실행 환경과 Repository를 함께 묶으면 위젯 트리 안에서 전역 플래그를 읽는 코드가 줄어든다.

```dart
enum AppMode { production, integrationTest }

class AppDependencies {
  final DeviceRepository devices;
  final MqttGateway mqtt;

  const AppDependencies({required this.devices, required this.mqtt});

  factory AppDependencies.create(AppMode mode) {
    if (mode == AppMode.integrationTest) {
      return AppDependencies(
        devices: FakeDeviceRepository(),
        mqtt: FakeMqttGateway(),
      );
    }
    return AppDependencies(
      devices: ApiDeviceRepository(),
      mqtt: AwsMqttGateway(),
    );
  }
}

Future<void> main({AppMode mode = AppMode.production}) async {
  WidgetsFlutterBinding.ensureInitialized();
  final deps = AppDependencies.create(mode);
  runApp(SmartHomeApp(dependencies: deps));
}
```

테스트 파일에서는 앱을 `integrationTest` 모드로 띄운다. `dart-define`보다 이 방식이 테스트 코드에서 의도가 분명하고, 실수로 운영 서버 URL을 넣을 위험도 낮았다.

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('보일러 전원을 켜면 제어 완료가 표시된다', (tester) async {
    await app.main(mode: AppMode.integrationTest);
    await tester.pumpAndSettle();

    await tester.tap(find.byKey(const Key('boiler_power_button')));
    await tester.pump(const Duration(milliseconds: 300));

    expect(find.text('켜짐'), findsOneWidget);
    expect(find.text('연결 중'), findsNothing);
  });
}
```

## Fake는 성공만 반환하면 부족하다

처음 만든 `FakeMqttGateway`는 `connect()`가 무조건 성공했다. 그러면 화면 전환은 검증되지만 재연결, 명령 중복, 타임아웃 분기는 전혀 검증되지 않는다. Fake 안에 상태와 이벤트를 넣어 실제 장애 조건도 재현할 수 있게 했다.

```dart
class FakeMqttGateway implements MqttGateway {
  bool connected = false;
  final commands = <String>[];

  @override
  Future<void> connect() async => connected = true;

  @override
  Future<void> publish(String topic, String payload) async {
    if (!connected) throw StateError('not connected');
    commands.add('$topic:$payload');
  }
}
```

다만 이 구조가 실제 API 응답 형식까지 보장해 주는 건 아니다. DTO 변경을 잡으려면 서버와 한 번 맞춰 보는 계약 테스트가 필요하다. 핵심 E2E 경로에서는 네트워크를 제거하고, 통신 포맷 검증은 별도 테스트로 두는 식으로 책임을 나눴다.

## CI에서 확인할 체크리스트

- 테스트 진입점이 `AppMode.production`을 사용하지 않는가
- Fake Repository에 고정된 집·기기·오류 응답이 있는가
- MQTT 연결 대기 대신 `pump`로 화면 상태를 제어하는가
- 테스트 종료 뒤 StreamSubscription과 Fake 서버 상태를 정리하는가

실제 네트워크를 붙인 E2E는 한두 개의 스모크 테스트로 제한하고, 나머지는 Fake 의존성으로 돌리는 편이 Firebase Test Lab 비용과 실행 시간을 함께 줄인다. 이번에 네트워크를 끊고 나니 테스트 하나가 18초에서 3초 안팎으로 줄었고, 서버 데이터 때문에 재실행하던 실패도 사라졌다.
