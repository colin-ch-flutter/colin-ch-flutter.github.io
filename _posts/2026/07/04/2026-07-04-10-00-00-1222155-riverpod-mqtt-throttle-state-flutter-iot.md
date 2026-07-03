---
layout: post
title: "Flutter IoT Riverpod throttle - MQTT 상태 업데이트를 화면 속도에 맞추기"
description: "Flutter IoT 앱에서 Riverpod으로 MQTT 상태 업데이트를 throttle 처리해 Flutter 스마트홈 대시보드 리빌드와 배터리 소모를 줄인 기준을 정리했다."
date: 2026-07-04
tags: [Flutter, Riverpod, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod MQTT throttle](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
이 그림에서는 MQTT 메시지를 그대로 화면에 밀어 넣지 않고, 앱이 처리할 수 있는 속도로 한 번 걸러야 한다는 점을 보면 된다.

Flutter IoT 앱에서 Flutter 스마트홈 대시보드는 MQTT 상태 업데이트를 모두 그릴 필요가 없다. Riverpod Provider 안에서 수신 스트림을 그대로 `watch`하면 데이터는 빠르게 반영되지만, 보일러 온도와 연결 품질이 1초에 여러 번 들어오는 순간 화면이 불필요하게 흔들린다. 내가 확인한 기준으론 제어 화면은 즉시성이 필요하고, 대시보드 요약 화면은 300~500ms 단위로 묶어도 사용자가 차이를 거의 느끼지 못했다.

처음엔 MQTT 메시지를 받는 즉시 `state = next`로 바꾸면 가장 정확하다고 생각했다. 실제로 로그만 보면 맞다. 문제는 화면이었다. 온도 21.1도, 21.2도, 21.1도가 짧게 반복될 때 카드 전체가 계속 리빌드됐고, BLE 연결 아이콘 애니메이션까지 같이 버벅였다. 정확한 상태와 보여줄 상태를 같은 것으로 본 게 실수였다.

## 어디에서 throttle을 걸지

내가 정한 기준은 단순했다. Repository는 원본 MQTT 메시지를 잃지 않고 받고, 화면용 Provider에서만 표시 속도를 조절한다. 그래야 장애 분석 로그와 UI 최적화가 섞이지 않는다.

| 위치 | 역할 | throttle 여부 |
|---|---|---|
| MQTT Repository | broker 수신, topic 파싱, 원본 로그 | 안 함 |
| Device State Provider | 최신 상태 보관, offline 판정 | 보통 안 함 |
| Dashboard View Provider | 카드 표시용 모델 생성 | 함 |
| Command Notifier | 사용자 제어 명령 publish | debounce와 별도 |

화면용 스트림에서 최신 값만 일정 간격으로 흘려보내는 코드다. 외부 패키지를 늘리기 싫어서 `Timer`로 처리했다.

```dart
final dashboardStateProvider =
    StreamProvider.autoDispose<DashboardViewState>((ref) {
  final repository = ref.watch(deviceStateRepositoryProvider);
  final controller = StreamController<DashboardViewState>();
  DashboardViewState? latest;
  Timer? timer;

  final sub = repository.watchDashboardState().listen((next) {
    latest = next;
    timer ??= Timer(const Duration(milliseconds: 400), () {
      final snapshot = latest;
      timer = null;
      if (snapshot != null && !controller.isClosed) {
        controller.add(snapshot);
      }
    });
  });

  ref.onDispose(() {
    timer?.cancel();
    sub.cancel();
    controller.close();
  });

  return controller.stream;
});
```

여기서 핵심은 첫 값을 늦게 보여주는 문제가 생길 수 있다는 점이다. 그래서 홈 진입 직후에는 로컬 캐시나 마지막 수신값을 즉시 보여주고, 이후 실시간 스트림만 400ms로 묶는 식이 낫다. 빈 화면에서 400ms를 기다리게 만들면 최적화가 아니라 체감 지연이 된다.

## debounce와 다르게 봐야 한다

버튼 연타를 막는 debounce는 마지막 사용자 의도만 보내는 데 초점이 있다. 반면 MQTT 수신 throttle은 서버나 기기가 이미 보낸 상태 중 화면에 필요한 빈도만 낮추는 작업이다. 둘을 같은 Notifier에 넣으면 코드가 빨리 꼬였다.

솔직하게 정리해보면 이렇다. 목표 온도 변경 명령은 사용자의 마지막 입력이 중요하다. 하지만 현재 온도, RSSI, 연결 품질 같은 값은 최신값이 중요하고 중간값 일부는 화면에서 버려도 된다. 나는 긴급 알림, 에러 코드, 도어락 열림 같은 이벤트성 메시지는 throttle 대상에서 제외했다. 한 번 빠뜨리면 안 되는 값이기 때문이다.

## 적용 기준

- 숫자 표시가 1초에 3회 이상 바뀌면 화면용 Provider에서 묶는다.
- 제어 결과, 에러, 알림은 묶지 않는다.
- 원본 MQTT 로그는 Repository 아래에서 그대로 남긴다.
- `autoDispose` Provider라면 `Timer`와 `StreamSubscription`을 반드시 정리한다.
- throttle 간격은 300ms부터 시작하고, 느리게 느껴지면 줄인다.

짧게 남기면 이렇다. Flutter IoT Riverpod 구조에서 MQTT 상태 업데이트를 모두 화면에 반영하는 건 정확해 보이지만 항상 좋은 선택은 아니었다. Flutter 스마트홈 대시보드는 최신 상태를 보여주되, 사람이 볼 수 있는 속도로 정리해서 그리는 편이 안정적이다. 원본 수신과 화면 표시를 분리하면 리빌드, 배터리, 디버깅 로그를 각각 다룰 수 있다.
