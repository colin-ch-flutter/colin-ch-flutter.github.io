---
layout: post
title: "Flutter integration_test 네트워크 재시도 - timeout과 flaky 테스트 분리"
description: "Flutter integration_test에서 MQTT·REST 응답 지연으로 flaky 테스트가 생길 때 무작정 재시도하지 않고, 요청 timeout·조건부 retry·실패 로그를 분리하는 방법을 정리했다."
date: 2026-08-30
tags: [Flutter, Dart, 테스트, integration_test, MQTT, IoT, CI/CD]
comments: true
share: true
---

![Flutter integration_test 네트워크 timeout과 retry 흐름](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 그림에서 볼 부분은 앱과 서버 사이의 지연이 테스트 실패로 바로 이어지는 구조다.

Flutter integration_test에서 MQTT 명령을 보내고 REST 응답을 기다리는 테스트는 `timeout`을 길게 잡는다고 안정되지 않는다. 네트워크가 느린 것과 서버가 실제로 실패한 것은 다른데, 이 둘을 같은 예외로 처리하면 CI에서 flaky 테스트가 된다. 내가 정리한 기준은 **요청별 timeout을 두고, 재시도 가능한 오류만 제한적으로 retry하며, 최종 실패 때는 원인을 남기는 것**이다.

## timeout만 늘렸을 때 생긴 문제

처음에는 MQTT ACK 대기 시간을 5초에서 30초로 늘렸다. 로컬 실기기에서는 통과율이 올라갔지만 CI에서 테스트 하나가 멈추면 전체 잡이 30초씩 늦어졌다. 브로커가 끊긴 경우에도 같은 메시지를 다시 보내는 문제까지 생겼다.

| 상황 | 단순 timeout 처리 | 분리한 처리 |
|---|---|---|
| 서버 응답이 늦음 | 긴 대기 후 실패 | 짧은 요청 timeout 후 제한 retry |
| HTTP 400·401 | 같은 요청 반복 | 즉시 실패, 인증 로직으로 전달 |
| MQTT 연결 끊김 | ACK를 계속 기다림 | 연결 상태 복구 후 명령 재전송 |
| 테스트 코드 오류 | 네트워크 오류처럼 보임 | 단계·requestId·마지막 상태 기록 |

모든 실패를 없애는 retry는 필요 없다. `400`, `401`, 잘못된 payload는 재시도하지 않고 그대로 실패해야 한다.

## 요청 timeout과 retry를 분리한다

아래 정책 객체는 REST와 MQTT 테스트가 같은 timeout 기준을 쓰게 한다.

```dart
class RetryPolicy {
  const RetryPolicy({this.maxAttempts = 3, this.timeout = const Duration(seconds: 4)});

  final int maxAttempts;
  final Duration timeout;
}

Future<T> withRetry<T>(
  Future<T> Function() request, {
  required RetryPolicy policy,
  bool Function(Object error)? retryWhen,
}) async {
  Object? lastError;

  for (var attempt = 1; attempt <= policy.maxAttempts; attempt++) {
    try {
      return await request().timeout(policy.timeout);
    } catch (error) {
      lastError = error;
      final canRetry = retryWhen?.call(error) ?? false;
      if (!canRetry || attempt == policy.maxAttempts) rethrow;
      await Future<void>.delayed(Duration(milliseconds: 300 * attempt));
    }
  }

  throw StateError('retry failed: $lastError');
}
```

`retryWhen`에는 socket disconnect나 5xx만 넣는다. 인증 실패와 validation 오류는 제외한다. 반복 대기 시간이 테스트를 늦추면 delay 함수를 주입해 가상 시간으로 바꾼다.

## integration_test에서는 성공 조건을 명시한다

UI에 “명령 전송 완료”가 표시됐다고 성공 처리하면 안 된다. MQTT는 publish 성공과 기기 처리 완료 사이에 시간이 있다. `commandId`가 같은 ACK인지 확인해야 다른 테스트의 메시지를 잘못 읽지 않는다.

```dart
testWidgets('네트워크 지연 후 명령이 한 번만 적용된다', (tester) async {
  final mqtt = FakeMqttService(
    ackDelay: const Duration(milliseconds: 800),
  );
  await startTestApp(tester, mqtt: mqtt);

  await tester.tap(find.text('켜기'));
  await tester.pump();

  await waitFor(tester, () => find.text('보일러 켜짐').evaluate().isNotEmpty,
      timeout: const Duration(seconds: 6));

  expect(mqtt.publishCount, 1);
  expect(mqtt.lastAckCommandId, mqtt.lastPublishedCommandId);
});
```

`waitFor`의 6초는 테스트 전체 제한이고 MQTT ACK timeout 4초와는 별개다. 이 경계를 섞으면 assertion은 통과해도 남은 Timer 때문에 프로세스가 종료되지 않을 수 있다. `tearDown`에서는 구독 해제와 Fake broker 종료를 보장한다.

retry가 끝난 뒤 `expected true`만 출력하면 원인을 찾기 어렵다. 테스트 계정, `commandId`, 시도 횟수, 마지막 MQTT 상태를 남기되 JWT와 운영 토픽은 기록하지 않는다. 내 기준은 REST 4초·2~3회, MQTT ACK 6초·1회 대기다.

정리하면 다음 세 가지다.

- timeout은 요청 단위와 테스트 전체 단위로 나눈다.
- retry는 네트워크 단절과 5xx처럼 결과가 바뀔 가능성이 있는 오류에만 적용한다.
- integration_test 성공 조건은 화면 문구가 아니라 `commandId`, ACK, publish 횟수까지 검증한다.
