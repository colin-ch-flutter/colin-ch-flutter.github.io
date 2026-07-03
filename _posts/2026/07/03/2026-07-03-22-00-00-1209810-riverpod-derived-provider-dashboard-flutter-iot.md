---
layout: post
title: "Flutter IoT Riverpod derived provider - Flutter 스마트홈 화면 모델 합치기"
description: "Flutter IoT 앱에서 Riverpod derived provider로 Flutter BLE 연결 상태와 MQTT 기기 상태를 화면 모델로 합치는 기준을 정리했다."
date: 2026-07-03
tags: [Flutter, Riverpod, MQTT, BLE, 스마트홈]
comments: true
share: true
---
![Flutter IoT Riverpod derived provider](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)
이 그림에서는 여러 스마트홈 기기 상태가 한 화면에 모일 때 원천 상태와 화면 표시 상태를 분리해야 한다는 점을 보면 된다.

Flutter IoT 화면은 Riverpod derived provider로 Flutter BLE 연결 상태와 MQTT 기기 상태를 합쳐서 보여주는 편이 낫다. Flutter 스마트홈 앱에서 화면이 필요한 값은 보일러 전원 하나가 아니라 연결 여부, 마지막 수신 시각, 명령 대기 상태, 오프라인 여부까지 섞인 결과다. 이 조합을 Widget 안에서 매번 만들면 화면 파일이 금방 지저분해진다.

처음엔 `ConsumerWidget` 안에서 `ref.watch()`를 3개 호출하고 바로 if문으로 처리했다. 해보니 됐다. 문제는 기기가 8개로 늘었을 때부터였다. BLE는 연결됨인데 MQTT는 12초째 메시지가 없고, 사용자는 전원 버튼을 방금 눌렀고, 캐시는 어제 값인 상황이 한 카드 안에 같이 들어왔다. 그때부터 화면은 상태를 그리는 곳이지 상태를 해석하는 곳이 아니라고 기준을 바꿨다.

| 구분 | 원천 Provider | derived provider에서 할 일 |
|---|---|---|
| BLE | 연결, 스캔, 권한 | 로컬 연결 가능 여부 판단 |
| MQTT | shadow, telemetry | 최신 기기 상태와 stale 여부 계산 |
| Command | 전송 중, 실패 | 버튼 비활성화와 에러 문구 결정 |
| Cache | 마지막 저장값 | 초기 표시값과 갱신 시각 보강 |

아래 코드는 화면에서 바로 쓰는 `DashboardCardState`를 별도 Provider로 만든 예시다.

```dart
final dashboardCardProvider =
    Provider.family<DashboardCardState, String>((ref, deviceId) {
  final ble = ref.watch(bleConnectionProvider(deviceId));
  final mqtt = ref.watch(mqttDeviceStateProvider(deviceId));
  final command = ref.watch(commandStateProvider(deviceId));
  final cached = ref.watch(cachedDeviceStateProvider(deviceId));

  final source = mqtt.valueOrNull ?? cached;
  final lastSeen = source?.updatedAt;
  final isStale = lastSeen == null
      || DateTime.now().difference(lastSeen) > const Duration(seconds: 10);

  return DashboardCardState(
    title: source?.name ?? '기기',
    powerOn: source?.powerOn ?? false,
    connectionLabel: ble.isConnected && !isStale ? '온라인' : '확인 필요',
    buttonEnabled: ble.isConnected && command.isSending == false,
    warningText: isStale ? '최근 상태가 아닐 수 있음' : null,
  );
});
```

여기서 핵심은 derived provider가 새 데이터를 가져오지 않는다는 점이다. Repository 호출, BLE reconnect, MQTT subscribe는 원천 Provider에서 끝낸다. derived provider는 이미 들어온 값을 조합해서 화면이 이해하기 쉬운 형태로 바꾼다. 이 선을 넘기면 테스트가 어려워지고, 리빌드 타이밍도 추적하기 힘들어진다.

내가 확인한 기준으론 `DateTime.now()`도 조심해야 한다. 위 예시는 설명을 위해 간단히 썼지만, 실제 테스트에서는 clock provider를 따로 두는 편이 낫다. 그래야 10초 stale 판정을 fake time으로 고정할 수 있다. 안 그러면 CI에서 9.8초와 10.1초 사이를 오가며 테스트가 가끔 깨진다.

주의할 점은 화면 모델을 너무 크게 만들지 않는 것이다. 대시보드 전체 상태를 하나의 거대한 provider로 합치면 카드 하나의 MQTT 값만 바뀌어도 전체 화면이 다시 계산된다. 나는 기기 카드 단위는 `family(deviceId)`로 쪼개고, 상단 요약처럼 전체 합계가 필요한 값만 별도 provider로 만들었다.

짧게 정리하면 이렇다.

- Widget은 상태를 해석하지 말고 이미 정리된 화면 모델을 그린다.
- BLE, MQTT, 캐시, command 상태는 원천 Provider로 분리한다.
- derived provider는 조합과 표시 판단까지만 맡긴다.
- 시간 판정은 테스트를 위해 clock provider로 빼는 편이 낫다.
