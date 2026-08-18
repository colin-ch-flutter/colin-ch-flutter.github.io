---
layout: post
title: "Flutter GetX Bindings 테스트 - IoT 의존성 주입을 Fake로 교체하기"
description: "Flutter GetX Bindings에서 Get.put·Get.lazyPut·Get.replace를 테스트용 Fake 서비스로 교체하고, MQTT와 BLE가 실제 연결 없이 검증되도록 만든 패턴을 정리했다."
date: 2026-08-18
tags: [Flutter, Dart, GetX, IoT, MQTT, BLE, 테스트, CI/CD]
comments: true
share: true
---

![Flutter GetX Bindings 의존성 주입 테스트와 IoT Fake 서비스 교체](/assets/images/flutter-getx-bindings-dependency-test-iot.png)

이 그림에서 봐야 할 부분은 앱이 운영용 MQTT·BLE 서비스를 그대로 쓰는 대신, 테스트 진입점에서 Fake 구현으로 갈아타는 흐름이다.

Flutter GetX Bindings는 화면에 필요한 의존성을 한곳에서 등록할 수 있어 편하다. 문제는 테스트에서 드러났다. `Get.put()`으로 등록한 실제 Repository가 생성되는 순간 MQTT 클라이언트와 BLE 플러그인을 건드렸고, 테스트는 화면을 그리기도 전에 죽었다. 처음엔 테스트 시작 전에 `Get.reset()`만 호출하면 된다고 생각했는데, 해보니 이미 생성된 객체를 지우는 것과 테스트용 객체를 주입하는 것은 별개의 문제였다.

## 운영 Binding과 테스트 Binding을 분리한다

운영 코드의 Binding 안에서 `if (isTest)`를 계속 늘리면 환경 분기가 화면 의존성까지 번진다. Binding 자체를 인터페이스로 묶고, 앱 진입점에서 구현체를 선택하는 편이 추적하기 쉽다.

```dart
abstract class HomeBinding {
  void register();
}

class ProductionHomeBinding implements HomeBinding {
  @override
  void register() {
    Get.lazyPut<MqttService>(() => AwsMqttService());
    Get.lazyPut<BleService>(() => FlutterBleService());
    Get.lazyPut<HomeRepository>(() => HomeRepository(
          mqtt: Get.find<MqttService>(),
          ble: Get.find<BleService>(),
        ));
    Get.lazyPut<HomeController>(() => HomeController(
          repository: Get.find<HomeRepository>(),
        ));
  }
}
```

`lazyPut`을 쓴 이유는 Binding이 실행되는 순간 플러그인을 초기화하지 않기 위해서다. 다만 화면이 `HomeController`를 찾는 순간에는 실제 서비스까지 연쇄적으로 만들어진다. 테스트에서는 그 지점을 Fake로 바꿔야 한다.

## Get.replace는 등록 후 교체할 때만 쓴다

`Get.replace`는 같은 타입의 인스턴스가 이미 등록되어 있을 때 유용하다. 반대로 아직 등록되지 않은 타입을 `replace`하려고 하면 테스트가 실패하므로, 테스트용 Binding은 등록 순서를 명시하는 편이 안전하다.

```dart
class FakeMqttService implements MqttService {
  final publishedTopics = <String>[];

  @override
  Future<void> connect() async {}

  @override
  Future<void> publish(String topic, String payload) async {
    publishedTopics.add(topic);
  }
}

class TestHomeBinding implements HomeBinding {
  final fakeMqtt = FakeMqttService();

  @override
  void register() {
    Get.put<MqttService>(fakeMqtt);
    Get.put<BleService>(FakeBleService());
    Get.put<HomeRepository>(HomeRepository(
          mqtt: Get.find<MqttService>(),
          ble: Get.find<BleService>(),
        ));
    Get.put<HomeController>(HomeController(
          repository: Get.find<HomeRepository>(),
        ));
  }
}

void setUpTestDependencies() {
  Get.reset();
  TestHomeBinding().register();
}
```

실제 앱이 먼저 등록되는 구조라면 아래처럼 교체할 수 있다. 이때 `Get.replace`가 기존 객체를 dispose하는 설정인지 확인해야 한다. 테스트에서 공유 Fake를 재사용하면 이전 테스트의 발행 기록이 남는 실수를 하기 쉽다.

```dart
void replaceWithFakes(FakeMqttService mqtt) {
  Get.replace<MqttService>(mqtt);
  Get.replace<BleService>(FakeBleService());
}
```

## Controller가 Fake를 사용했는지 검증한다

코드 바로 위에 테스트 시나리오를 적어두면 이 테스트가 연결 성공을 검증하는지, 명령 발행만 검증하는지 구분할 수 있다. 여기서는 실제 네트워크 없이 보일러 전원 명령의 토픽만 확인한다.

```dart
test('보일러 전원 토글은 Fake MQTT에 한 번만 발행된다', () async {
  final binding = TestHomeBinding();
  Get.reset();
  binding.register();

  final controller = Get.find<HomeController>();
  await controller.toggleBoiler('living-room', true);

  expect(binding.fakeMqtt.publishedTopics,
      contains('home/living-room/boiler/command'));
  expect(binding.fakeMqtt.publishedTopics, hasLength(1));
});
```

여기서 Binding을 두 번 생성하거나 `register()`를 중복 호출하면 의존성이 덮어써질 수 있다. 실제 테스트에서는 Binding을 하나만 생성하고, Fake 참조도 같은 객체에서 꺼내야 한다. 테스트 코드가 짧아 보여도 GetX 전역 컨테이너가 섞이면 순서에 따라 통과하거나 실패하는 플래키 테스트가 된다.

| 상황 | 등록 방식 | 확인할 점 |
|---|---|---|
| 운영 앱 시작 | `Get.lazyPut` | 플러그인 지연 초기화 |
| 테스트 전용 시작 | `Get.put` | Fake 인스턴스 참조 확보 |
| 이미 등록된 객체 교체 | `Get.replace` | 기존 타입 등록 여부와 dispose |
| 테스트 종료 | `Get.reset` | 전역 컨테이너와 Stream 정리 |

## 실패했던 지점과 기준

`Get.reset()`을 `tearDown`에만 넣고 `setUp`에는 넣지 않았을 때 이전 테스트의 Controller가 다음 테스트에서 재사용됐다. 또 `Get.lazyPut(..., fenix: true)`를 운영 설정 그대로 테스트에 복사했더니, 삭제한 Fake가 다시 생성되면서 실제 의존성이 섞였다. 테스트 Binding에서는 `fenix`를 기본값으로 두지 않는 게 낫다.

CI에서는 테스트마다 다음 순서를 고정했다.

1. `Get.reset()`으로 전역 컨테이너를 비운다.
2. Fake BLE와 Fake MQTT를 등록한다.
3. Repository와 Controller를 Fake 기준으로 만든다.
4. 테스트 종료 후 Stream 구독과 타이머를 닫는다.

짧게 정리하면 Flutter GetX Bindings 테스트의 핵심은 `Get.replace` 사용법 자체가 아니다. 운영 의존성을 Binding 경계 밖으로 밀어내고, 테스트가 시작될 때 Fake 그래프를 처음부터 조립하는 데 있다. 그러면 BLE 권한, MQTT 브로커 상태, 네트워크 지연과 무관하게 IoT Controller의 명령·실패·재시도 로직을 반복해서 검증할 수 있다.
