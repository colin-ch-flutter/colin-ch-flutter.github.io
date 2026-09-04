---
layout: post
title: "Flutter IoT MQTT 계약 테스트 - 기기 JSON payload 불일치를 CI에서 잡는 방법"
description: "Flutter IoT MQTT 앱에서 펌웨어와 JSON payload 계약이 어긋나 생기는 MQTT 연결 끊김과 런타임 오류를 Dart 계약 테스트로 CI에서 잡는 방법을 정리했다."
date: 2026-09-04
tags: [Flutter, Dart, MQTT, IoT, 테스트, CI/CD, 스마트홈]
comments: true
share: true
---

![Flutter IoT MQTT payload 계약 테스트와 CI 검증 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

*이 그림에서 볼 것은 MQTT 브로커 자체보다, 오가는 메시지의 형식을 앱과 기기가 함께 지켜야 한다는 점이다.*

Flutter IoT 앱에서 MQTT 연결 끊김을 추적하다 보면 브로커가 아니라 JSON payload가 원인인 경우가 있다. 펌웨어가 `temperature`를 숫자 대신 문자열로 보내거나, 앱이 기대한 `reportedAt` 필드를 빼고 보내는 식이다. 처음엔 `try/catch`로 파싱 예외만 막으면 되는 줄 알았는데 아니었다. 화면은 멈추지 않아도 상태 갱신이 조용히 누락됐다.

## 문제는 연결 테스트로 잡히지 않는다

기존 MQTT 연결 테스트는 연결 성공, 구독, publish 여부를 확인한다. 하지만 메시지 안쪽의 계약까지 보장하지는 않는다. 특히 펌웨어 버전이 여러 개 공존하는 IoT 환경에서는 “연결됐다”와 “해석할 수 있다”가 전혀 다른 상태다.

| 상황 | 앱에서 보이는 결과 | 필요한 검증 |
|---|---|---|
| `temperature: 23.4` | 정상 갱신 | 숫자 타입 확인 |
| `temperature: "23.4"` | 런타임 파싱 실패 | 타입 계약 테스트 |
| `deviceId` 누락 | 다른 기기 상태로 매핑될 위험 | 필수 키 테스트 |
| `schemaVersion: 2` | 구버전 앱에서 알 수 없는 필드 | 버전 호환성 테스트 |

그래서 나는 MQTT service 테스트와 별개로, 앱이 수신하는 메시지 샘플을 고정해두는 계약 테스트를 추가했다. 네트워크나 실제 브로커가 없어도 실행된다.

## Dart 모델에서 계약을 한 곳에 모으기

파싱 로직이 화면이나 Controller에 흩어지면 테스트 대상이 커진다. 수신 payload를 모델로 변환하는 함수가 실패를 명확히 알리도록 만들었다.

```dart
import 'dart:convert';

class DeviceTelemetry {
  const DeviceTelemetry({
    required this.deviceId,
    required this.temperature,
    required this.isHeating,
    required this.schemaVersion,
  });

  final String deviceId;
  final double temperature;
  final bool isHeating;
  final int schemaVersion;

  factory DeviceTelemetry.fromMqtt(String raw) {
    final value = jsonDecode(raw);
    if (value is! Map<String, dynamic>) {
      throw const FormatException('payload must be a JSON object');
    }

    final deviceId = value['deviceId'];
    final temperature = value['temperature'];
    final isHeating = value['isHeating'];
    final schemaVersion = value['schemaVersion'];

    if (deviceId is! String ||
        temperature is! num ||
        isHeating is! bool ||
        schemaVersion is! int) {
      throw const FormatException('telemetry payload contract mismatch');
    }

    return DeviceTelemetry(
      deviceId: deviceId,
      temperature: temperature.toDouble(),
      isHeating: isHeating,
      schemaVersion: schemaVersion,
    );
  }
}
```

여기서 `num`을 받은 뒤 `double`로 변환한 이유는 JSON 숫자가 정수로 들어오는 보일러 초기 상태도 허용하기 위해서다. 반대로 `temperature is! num`처럼 타입은 엄격하게 검사했다. 문자열을 자동 변환하면 잘못된 펌웨어가 배포돼도 테스트가 통과한다.

## 샘플 payload를 계약으로 고정하기

테스트 파일에는 실제 장치가 보내는 대표 메시지와 고장 사례를 함께 둔다. 성공 케이스 하나만 작성하면 필드가 빠진 메시지를 놓치기 쉽다.

```dart
import 'package:test/test.dart';

void main() {
  group('DeviceTelemetry MQTT contract', () {
    test('정상 payload를 모델로 변환한다', () {
      final telemetry = DeviceTelemetry.fromMqtt('''
        {"deviceId":"boiler-01","temperature":23.5,
         "isHeating":true,"schemaVersion":1}
      ''');

      expect(telemetry.deviceId, 'boiler-01');
      expect(telemetry.temperature, 23.5);
      expect(telemetry.isHeating, isTrue);
    });

    test('필수 필드 타입이 바뀌면 즉시 실패한다', () {
      expect(
        () => DeviceTelemetry.fromMqtt('''
          {"deviceId":"boiler-01","temperature":"23.5",
           "isHeating":true,"schemaVersion":1}
        '''),
        throwsA(isA<FormatException>()),
      );
    });

    test('필수 필드가 빠지면 실패한다', () {
      expect(
        () => DeviceTelemetry.fromMqtt(
          '{"deviceId":"boiler-01","temperature":23.5}',
        ),
        throwsA(isA<FormatException>()),
      );
    });
  });
}
```

실제 프로젝트에서는 이 샘플을 `test/fixtures/mqtt/telemetry_v1.json`으로 분리하고, 펌웨어 팀이 payload를 바꿀 때 fixture와 계약 테스트를 함께 수정한다. 테스트가 실패하면 “MQTT가 안 된다”가 아니라 “v1의 `temperature` 타입이 바뀌었다”라고 원인을 좁혀 말할 수 있다.

## CI에서는 브로커 없이 실행한다

계약 테스트는 외부 서비스에 의존하지 않으므로 일반 Unit Test 단계에 넣는다. 브로커 주소나 AWS 인증서가 없어도 된다.

```yaml
- name: Run MQTT contract tests
  run: dart test test/contracts/mqtt_payload_contract_test.dart
```

주의할 점은 호환성 정책이다. 필드를 무조건 엄격하게 검사하면 펌웨어가 새 선택 필드를 추가할 때 앱 테스트가 깨진다. 필수 필드와 타입은 엄격하게, 알 수 없는 추가 필드는 무시하는 식으로 경계를 정해야 한다. `schemaVersion`이 올라가는 경우에는 구버전 앱이 안전하게 무시할지, 명시적인 `UnsupportedSchemaException`을 낼지도 계약에 적어둬야 한다.

처음엔 모든 MQTT topic을 하나의 거대한 JSON 모델로 합쳤다. 해보니 보일러 상태, 에너지 사용량, 알림 이벤트가 서로 다른 변경 주기를 갖고 있어 수정 때마다 관련 없는 테스트까지 깨졌다. topic별 payload와 버전을 나누는 편이 유지보수하기 훨씬 편했다.

## 짧게 정리하면

- MQTT 연결 성공 테스트만으로는 payload 불일치를 잡을 수 없다.
- 앱 경계에서 필수 키와 타입을 검사하고 `FormatException`으로 실패시킨다.
- 정상·타입 변경·필드 누락 샘플을 fixture로 고정한다.
- 브로커 없는 계약 테스트를 CI에 넣으면 펌웨어 변경을 빠르게 발견할 수 있다.
- 새 필드 추가와 기존 필드 타입 변경은 호환성 정책을 다르게 다뤄야 한다.

기존 `mqtt5_client` 연결 로직을 Fake 서비스로 격리한 테스트와 함께 적용하면, 통신 연결 문제와 payload 형식 문제를 서로 다른 실패로 분리할 수 있다.

- [{% post_url 2026-06-24-14-00-00-666630-mqtt5-client-unit-test-fake-service-isolation %}]({% post_url 2026-06-24-14-00-00-666630-mqtt5-client-unit-test-fake-service-isolation %}) — `mqtt5_client` Fake 서비스 격리
