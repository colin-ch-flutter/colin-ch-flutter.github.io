---
layout: post
title: "IoT의 심장, MQTT 프로토콜 이해하기"
description: " "
date: 2026-05-12
tags: [MQTT, IoT, 프로토콜, 메시징]
comments: true
share: true
---
![네트워크 통신](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

## MQTT가 뭔지부터

MQTT(Message Queuing Telemetry Transport)는 1999년에 IBM이 만든 경량 메시지 프로토콜이다. 원유 파이프라인을 위성 링크로 모니터링하기 위해 만들어진 것인데, 그 특성이 IoT에도 딱 맞아서 널리 쓰이게 됐다.

핵심 특징:
- **Publish/Subscribe 패턴**: 보내는 쪽과 받는 쪽이 직접 연결되지 않음
- **경량**: 헤더가 최소 2바이트로 매우 작음
- **저대역폭 환경 최적화**: 느린 네트워크, 불안정한 연결에서도 동작
- **브로커 중심**: 모든 메시지가 브로커를 거쳐 전달됨

## Publish/Subscribe 패턴

HTTP는 클라이언트가 서버에 직접 요청한다. MQTT는 다르다.

```
[IoT 기기 기기] ---publish---> [MQTT 브로커] ---deliver---> [앱]
[앱] ---------publish---> [MQTT 브로커] ---deliver---> [IoT 기기 기기]
```

기기와 앱이 브로커를 통해 통신한다. 서로의 존재를 몰라도 된다. 기기는 "나 온도가 22도야" 하고 토픽에 발행하고, 앱은 그 토픽을 구독해서 데이터를 받는다.

이 구조의 장점은 확장성이다. 기기가 100대로 늘어도 앱은 여전히 브로커 한 곳만 바라보면 된다.

## Topic 구조

MQTT 토픽은 슬래시로 구분되는 계층 구조다:

```
smarthome/device/ABC123/status       # 기기 ABC123의 상태
smarthome/device/ABC123/command      # 기기 ABC123에 대한 명령
smarthome/user/USER001/notification  # 사용자 USER001에 대한 알림
```

와일드카드:
- `+`: 한 레벨 대체 (예: `smarthome/device/+/status` → 모든 기기의 상태)
- `#`: 이 레벨 이하 모두 (예: `smarthome/device/ABC123/#` → ABC123 기기의 모든 토픽)

## QoS(Quality of Service)

MQTT는 세 가지 QoS를 지원한다:

| QoS | 설명 | 전달 보장 | 중복 가능성 |
|-----|------|-----------|-------------|
| 0 | At most once (최대 1번) | 없음 | 없음 |
| 1 | At least once (최소 1번) | 있음 | 있음 |
| 2 | Exactly once (정확히 1번) | 있음 | 없음 |

IoT 기기 제어 명령은 QoS 1을 쓴다. 전달은 보장되어야 하고, 중복은 기기가 무시하도록 처리한다.

상태 업데이트는 QoS 0으로 충분하다. 최신 상태가 주기적으로 오기 때문에 하나 놓쳐도 괜찮다.

## Retain 메시지

Retain 플래그를 설정하면 브로커가 마지막 메시지를 저장해둔다. 새로운 구독자가 생기면 저장된 메시지를 즉시 받는다.

기기 상태 토픽에 Retain을 쓰면 앱이 처음 구독할 때 마지막 상태를 즉시 받을 수 있다. 기기에서 다음 상태 업데이트가 올 때까지 기다리지 않아도 된다.

## Last Will Testament (LWT)

MQTT에는 "유언" 기능이 있다. 연결할 때 "나 갑자기 끊어지면 이 메시지 발행해줘" 하고 브로커에 등록해두는 것이다.

기기가 예기치 않게 오프라인이 되면 브로커가 LWT 메시지를 발행해서 앱에 알려준다:

```dart
final willMessage = MqttConnectMessage()
    .withWillTopic('smarthome/device/$deviceId/status')
    .withWillMessage('{"online": false}')
    .withWillQos(MqttQos.atLeastOnce)
    .withWillRetain();
```

## MQTT vs HTTP 비교

왜 그냥 HTTP를 안 쓰는지 자주 받는 질문이다.

HTTP로 기기 상태를 실시간으로 받으려면 폴링을 해야 한다. 1초마다 "지금 온도 얼마야?" 하고 요청을 보내는 방식이다. 기기가 100대면 초당 100개 요청이다.

MQTT는 상태가 바뀔 때만 메시지가 온다. 훨씬 효율적이다.

반대로 기기 상태를 제어할 때는 앱이 명령을 보내고 기다리는 게 아니라, 토픽에 발행하고 응답 토픽을 구독한다. 비동기적으로 동작한다.

## MQTT5 vs MQTT3

mqtt5_client를 쓰는 이유도 간단히 정리한다. MQTT5는 MQTT3.1.1 대비 이런 기능이 추가됐다:

- **이유 코드**: 에러 원인을 더 자세히 알 수 있음
- **사용자 속성**: 메시지에 커스텀 메타데이터 추가 가능
- **공유 구독**: 여러 클라이언트가 같은 구독을 공유
- **메시지 만료**: 메시지에 유효 시간 설정 가능

AWS IoT Core가 MQTT5를 지원하기 때문에 더 풍부한 기능을 활용할 수 있다.

---

다음 편은 AWS IoT Core에 실제로 연결하는 방법이다.
