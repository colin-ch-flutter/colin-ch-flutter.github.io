---
layout: post
title: "Flutter IoT Riverpod Crashlytics - BLE와 MQTT 장애 로그를 운영에서 남기는 기준"
description: "Flutter IoT 앱에서 Riverpod ProviderObserver와 Crashlytics로 Flutter BLE 재연결, MQTT 연결 끊김 상태를 운영 로그에 남기는 기준을 정리했다."
date: 2026-07-14
tags: [Flutter, Riverpod, Firebase, Crashlytics, BLE, MQTT, IoT, 스마트홈]
comments: true
share: true
---
![Flutter IoT Riverpod Crashlytics 운영 로그](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)
이 그림에서는 운영 로그가 화면 캡처보다 장애 흐름을 더 잘 보여줘야 한다는 점을 봐야 한다.

Flutter IoT 앱에서 Riverpod 상태를 Crashlytics로 보낼 때는 모든 Provider를 남기면 안 됐다. Flutter BLE 재연결과 MQTT 연결 끊김처럼 장애 원인을 좁히는 상태만 남기고, 토큰·이메일·Space 이름 같은 값은 잘라내는 편이 운영에서 더 오래 버텼다. 처음엔 ProviderObserver 로그를 전부 `log()`로 보냈는데, 하루 테스트만 해도 노이즈가 너무 많았다.

## 운영 로그는 디버그 로그와 다르다

개발 중에는 `debugPrint()`가 편하다. BLE 스캔 시작, MQTT subscribe, 보일러 명령 publish를 줄줄이 찍으면 흐름이 보인다. 문제는 운영 앱이다. 사용자가 LTE에서 앱을 켰다가 닫고, 백그라운드 복귀 중 MQTT가 끊기고, BLE 권한이 거부되는 상황이 섞이면 로그가 금방 쓰레기가 된다.

내가 기준을 나눈 표는 이랬다.

| 상태 | 운영 로그 여부 | 이유 |
|---|---:|---|
| MQTT connected/disconnected | O | 장애 시간대와 직접 연결된다 |
| BLE scanning/connected/failed | O | 실기기 재현이 어렵다 |
| 버튼 pending | 제한적 | 중복 명령 분석에만 필요하다 |
| 폼 입력값 | X | 개인정보 위험이 크다 |
| JWT, refresh token | X | 절대 남기면 안 된다 |

## ProviderObserver에서 값부터 줄인다

이 코드는 Riverpod 상태 변경 중 운영에 보낼 이벤트만 골라 Crashlytics custom key와 log로 남기는 예시다.

```dart
class IotCrashlyticsObserver extends ProviderObserver {
  IotCrashlyticsObserver(this.crashlytics);

  final FirebaseCrashlytics crashlytics;

  @override
  void didUpdateProvider(
    ProviderObserverContext context,
    Object? previousValue,
    Object? newValue,
  ) {
    final name = context.provider.name;
    if (name == null || !_isOperationalProvider(name)) return;

    final sanitized = _sanitizeState(newValue);
    crashlytics.setCustomKey('riverpod_last_provider', name);
    crashlytics.setCustomKey('riverpod_last_state', sanitized);
    crashlytics.log('provider=$name state=$sanitized');
  }

  bool _isOperationalProvider(String name) {
    return name.startsWith('mqttSession') ||
        name.startsWith('bleConnection') ||
        name.startsWith('boilerCommand');
  }

  String _sanitizeState(Object? value) {
    final text = value.toString();
    final masked = text
        .replaceAll(RegExp(r'token=[^,)]*'), 'token=***')
        .replaceAll(RegExp(r'email=[^,)]*'), 'email=***');
    return masked.length > 120 ? masked.substring(0, 120) : masked;
  }
}
```

여기서 헷갈렸던 건 Crashlytics custom key를 히스토리처럼 쓸 수 있다고 생각한 부분이다. custom key는 최신 값을 보는 데 가깝다. 그래서 긴 흐름은 `log()`에 짧게 남기고, custom key에는 “최근에 흔들린 Provider”만 넣었다.

## 많이 남길수록 원인 찾기가 느려진다

운영 로그는 많다고 좋은 게 아니었다. MQTT reconnect가 1초 간격으로 반복될 때 모든 상태를 남기면 실제 원인인 `SigV4 403 오류`나 네트워크 전환 로그가 밀린다. BLE도 비슷했다. iOS 실기기에서 권한 거부와 주변 기기 미발견은 다른 문제인데, 둘 다 `scan failed`로만 남기면 나중에 구분이 안 됐다.

그래서 상태 이름은 거칠게라도 분리했다. `ble_permission_denied`, `ble_scan_timeout`, `mqtt_auth_failed`, `mqtt_socket_closed`처럼 앱에서 판단 가능한 수준까지만 쪼갰다. 서버 원인까지 앱에서 단정하지는 않았다. 내가 확인한 기준으론 이 정도가 로그 양과 분석 속도의 균형이 맞았다.

## 짧은 기준

- Flutter IoT 운영 로그는 Flutter BLE와 MQTT 상태 전이만 좁게 남긴다.
- Riverpod `ProviderObserver`는 모든 상태 덤프가 아니라 필터 역할로 쓴다.
- Crashlytics custom key에는 마지막 장애 단서만 넣고, 긴 흐름은 짧은 log로 남긴다.
- 토큰, 이메일, Space 이름, 기기 별칭은 마스킹하거나 아예 보내지 않는다.
- 운영 로그가 늘어날수록 장애 분석이 쉬워지는 게 아니라, 기준 없이 늘리면 더 느려진다.
