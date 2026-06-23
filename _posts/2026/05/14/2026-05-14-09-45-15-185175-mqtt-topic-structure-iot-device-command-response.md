---
layout: post
title: "MQTT 토픽 구조 설계 - IoT 기기 명령과 응답"
description: " "
date: 2026-05-14
tags: [MQTT, IoT, 토픽설계, 아키텍처]
comments: true
share: true
---

# MQTT 토픽 구조 설계 - IoT 기기 명령과 응답

![스마트홈 제어](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## 토픽 설계가 중요한 이유

MQTT 토픽 구조를 처음부터 잘 설계해두지 않으면 나중에 엄청난 고통이 따른다. 기기 펌웨어와 앱이 동시에 수정되어야 하는데, 이미 배포된 기기의 펌웨어를 전부 바꾸는 건 불가능에 가깝다.

처음 설계할 때 확장성을 고려해서 잡는 게 중요하다.

## 기본 토픽 구조

스마트홈 앱의 토픽 구조를 추상화하면 이런 형태다:

```
{prefix}/{version}/{deviceType}/{deviceId}/{direction}/{category}
```

예시:
```
smarthome/v1/iot_device/ABC123DEF456/shadow/update        # 앱→기기: 상태 변경 요청
smarthome/v1/iot_device/ABC123DEF456/shadow/update/delta  # 기기→앱: 실제 상태 변경됨
smarthome/v1/iot_device/ABC123DEF456/event/alert          # 기기→앱: 알람 발생
smarthome/v1/iot_device/ABC123DEF456/fota/notify          # 서버→기기: 펌웨어 업데이트 알림
```

## Command와 Response 분리

앱이 기기에 명령을 보내는 토픽과, 기기가 응답하는 토픽을 분리한다:

```
# 앱 → 기기 (명령)
smarthome/device/{deviceId}/command/power
smarthome/device/{deviceId}/command/temperature
smarthome/device/{deviceId}/command/mode

# 기기 → 앱 (응답/상태)
smarthome/device/{deviceId}/status
smarthome/device/{deviceId}/event
```

이렇게 분리하는 이유는 명령이 여러 앱(여러 가족 구성원)에서 올 수 있기 때문이다. 기기는 명령 토픽만 구독하고, 처리 결과를 상태 토픽에 발행한다.

## AWS Device Shadow 패턴

AWS IoT Core의 Device Shadow를 활용하면 더 강력하다:

```
# Shadow 업데이트 요청 (앱 → AWS)
$aws/things/{deviceId}/shadow/update

# Shadow 업데이트 성공 알림 (AWS → 앱)
$aws/things/{deviceId}/shadow/update/accepted

# Shadow에서 기기와 앱 상태 불일치 (AWS → 기기)
$aws/things/{deviceId}/shadow/update/delta
```

Shadow의 핵심은 `desired`와 `reported` 상태다:

```json
{
  "state": {
    "desired": {
      "power": "on",
      "temperature": 24
    },
    "reported": {
      "power": "off",
      "temperature": 20
    },
    "delta": {
      "power": "on",
      "temperature": 24
    }
  }
}
```

앱이 `desired`를 바꾸면, 기기는 `delta`를 받아서 실제 동작을 변경하고 `reported`를 업데이트한다. 기기가 오프라인이어도 `desired`는 저장되어 있다가 기기가 다시 온라인이 되면 전달된다.

## 여러 기기 관리

가족이 여러 기기를 등록한 경우, 앱은 기기별 토픽을 모두 구독한다:

```dart
void subscribeToDevices(List<String> deviceIds) {
  for (final deviceId in deviceIds) {
    _subscribe('smarthome/device/$deviceId/status', QoS.atMostOnce);
    _subscribe('smarthome/device/$deviceId/event', QoS.atLeastOnce);
  }
}
```

기기가 추가/삭제될 때 동적으로 구독을 추가/해제한다.

## 실수로 배운 것들

처음 설계할 때 버전 prefix를 빠뜨렸다. 나중에 프로토콜을 바꿔야 할 때 기존 기기와 새 기기를 동시에 지원하기 어려워졌다. 토픽에 항상 버전을 포함하는 게 좋다.

그리고 디바이스 타입도 토픽에 포함해야 한다. IoT 기기와 IoT 모니터링 대시보드 IoT 기기의 프로토콜이 다른데, 타입 없이 기기 ID만으로 구분하려면 런타임에 타입을 파악해야 해서 복잡해진다.

---

다음 편은 실제 mqtt5_client로 IoT 기기 On/Off 명령을 보내는 코드다.
