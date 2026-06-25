---
layout: post
title: "AWS IoT Core에 연결하기 - SigV4 서명 인증"
description: " "
date: 2026-05-13
tags: [Flutter, MQTT, AWS, IoTCore, SigV4, 인증]
comments: true
share: true
---
![클라우드 AWS](https://images.unsplash.com/photo-1451187580459-43490279c0fa?w=800&q=80)

## AWS IoT Core를 선택한 이유

MQTT 브로커는 직접 구축할 수도 있고(Mosquitto, EMQ X 등), 관리형 서비스를 쓸 수도 있다. AWS IoT Core를 선택한 이유는:

1. **확장성**: 수백만 기기 연결을 AWS가 알아서 처리
2. **보안**: 기기별 인증서, IAM 정책으로 세분화된 접근 제어
3. **Device Shadow**: 기기 오프라인 중 상태 저장 기능 내장
4. **타 AWS 서비스 연동**: Lambda, DynamoDB 등과 쉽게 연결

## 연결 방식: WebSocket + SigV4

AWS IoT Core는 두 가지 연결 방식을 지원한다:
- **X.509 인증서**: 기기 단에서 주로 사용
- **WebSocket + SigV4**: 앱에서 주로 사용

모바일 앱은 WebSocket + SigV4 방식을 쓴다. 인증서 파일을 앱에 번들하는 것보다 안전하고, IAM 자격증명으로 동적으로 인증할 수 있다.

## SigV4 서명이란

AWS API를 호출할 때 요청이 진짜 AWS 자격증명을 가진 사람이 보낸 것인지 검증하는 서명 방식이다.

WebSocket URL에 서명 정보를 쿼리 파라미터로 붙이는 형태다:

```
wss://{endpoint}/mqtt?X-Amz-Algorithm=AWS4-HMAC-SHA256
    &X-Amz-Credential=...
    &X-Amz-Date=...
    &X-Amz-Expires=...
    &X-Amz-SignedHeaders=host
    &X-Amz-Signature=...
```

이 URL을 직접 계산하려면 꽤 복잡한데, `aws_signature_v4` 패키지가 이걸 대신해준다.

## 패키지 설정

```yaml
dependencies:
  aws_signature_v4: ^0.6.10
  aws_common: ^0.7.12
```

## SigV4 서명된 WebSocket URL 생성

```dart
// data/services/iot_service.dart
class IotService {
  Future<String> buildMqttUrl({
    required String endpoint,      // xxx.iot.ap-northeast-2.amazonaws.com
    required String region,        // ap-northeast-2
    required String accessKeyId,
    required String secretAccessKey,
    required String sessionToken,  // 임시 자격증명 사용 시
  }) async {
    final signer = AWSSigV4Signer(
      credentialsProvider: AWSCredentialsProvider(
        AWSCredentials(
          accessKeyId,
          secretAccessKey,
          sessionToken,
        ),
      ),
    );

    final scope = AWSCredentialScope(
      region: region,
      service: AWSService('iotdevicegateway'),
    );

    final request = AWSHttpRequest.get(
      Uri.parse('wss://$endpoint/mqtt'),
      headers: {'host': endpoint},
    );

    final signedRequest = await signer.sign(
      request,
      credentialScope: scope,
    );

    return signedRequest.uri.toString();
  }
}
```

## MQTT 연결

생성한 URL로 MQTT 연결:

```dart
// data/services/ble_service.dart → IotService 활용
Future<void> connect(MqttContext context) async {
  final mqttUrl = await _iotService.buildMqttUrl(
    endpoint: context.endpoint,
    region: context.region,
    accessKeyId: context.credentials.accessKeyId,
    secretAccessKey: context.credentials.secretAccessKey,
    sessionToken: context.credentials.sessionToken,
  );

  final client = MqttServerClient.withPort(
    context.endpoint,
    context.clientId,
    443,
  );

  client.secure = true;
  client.useWebSocket = true;
  client.websocketProtocols = ['mqtt'];
  client.port = 443;

  final connMessage = MqttConnectMessage()
      .withClientIdentifier(context.clientId)
      .withWillQos(MqttQos.atLeastOnce)
      .keepAliveFor(60)
      .startClean();

  client.connectionMessage = connMessage;

  await client.connect();
}
```

## 자격증명 갱신

AWS STS(Security Token Service)에서 발급하는 임시 자격증명은 만료 시간이 있다. 만료 전에 갱신해야 MQTT 연결을 유지할 수 있다:

```dart
// 자격증명 만료 15분 전에 갱신
Timer.periodic(const Duration(minutes: 1), (_) async {
  if (_credentials.isExpiringSoon(const Duration(minutes: 15))) {
    final newCredentials = await _refreshCredentials();
    await _reconnectWithNewCredentials(newCredentials);
  }
});
```

## 디버깅 팁

연결 실패 시 디버깅이 쉽지 않다. 유용한 팁 몇 가지:

1. **SigV4 서명 검증**: AWS CLI로 같은 자격증명으로 연결이 되는지 먼저 확인
2. **IoT Policy 확인**: IAM 정책에 `iot:Connect`, `iot:Publish`, `iot:Subscribe`, `iot:Receive` 권한이 있는지 확인
3. **로그 활성화**:
```dart
client.logging(on: true); // mqtt_client 로그 켜기
```

---

다음 편은 MQTT 토픽 구조를 어떻게 설계했는지다.
