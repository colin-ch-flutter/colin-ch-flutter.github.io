---
layout: post
title: "Flutter IoT AsyncValue - Flutter 스마트홈 대시보드 로딩과 에러 상태 정리"
description: "Flutter IoT 앱에서 Riverpod AsyncValue로 Flutter BLE와 MQTT 상태를 대시보드에 안정적으로 표시하는 방법을 실제 코드 기준으로 정리했다."
date: 2026-07-02
tags: [Flutter, IoT, BLE, MQTT, 스마트홈]
comments: true
share: true
---
![Flutter IoT 스마트홈 대시보드 상태 관리](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

Flutter IoT 대시보드는 `loading`, `error`, `data`를 화면마다 따로 만들지 말고 `AsyncValue` 하나로 밀어붙이는 편이 낫다. Flutter 스마트홈 앱에서 Flutter BLE 연결 상태와 MQTT 수신 상태를 같이 보여주다 보면, 로딩 플래그가 두세 개로 늘어나는 순간부터 화면이 거짓말을 하기 시작한다.

[GetX 상태관리]({% post_url 2026-05-02-09-21-39-037035-getx-state-management-routing-flutter-iot %})를 쓸 때는 `isLoading`, `errorMessage`, `deviceStatus`를 Controller에 따로 뒀다. 처음엔 보기 쉬웠다. 근데 BLE 재연결 중인데 MQTT 값은 이전 데이터를 보여줘야 하는 상황에서 꼬였다. 전체 화면을 로딩으로 덮으면 사용자는 보일러가 꺼진 줄 알고, 이전 데이터를 그대로 두면 연결이 살아있는 것처럼 보였다.

Riverpod으로 옮기면서 기준을 바꿨다. 상태는 하나의 값으로 만들고, UI는 그 값이 아직 준비 중인지, 실패했는지, 데이터가 있는지만 본다.

```dart
final dashboardProvider =
    FutureProvider.autoDispose<DashboardState>((ref) async {
  final ble = ref.watch(bleConnectionProvider);
  final mqtt = ref.watch(mqttStatusProvider);
  final repository = ref.watch(deviceRepositoryProvider);

  return repository.loadDashboard(
    bleState: ble.valueOrNull,
    mqttState: mqtt.valueOrNull,
  );
});
```

여기서 중요한 건 `valueOrNull`이다. BLE나 MQTT가 잠깐 로딩이어도 대시보드 전체를 무조건 비우지 않는다. 최근 확보한 값이 있으면 그걸 기준으로 화면을 유지하고, 새 데이터가 들어오면 갱신한다. 실제 기기 앱에서는 이 차이가 크다. 네트워크가 1초 흔들렸다고 카드 전체가 깜빡이면 앱이 불안정해 보인다.

화면에서는 `AsyncValue.when`을 그대로 쓴다.

```dart
class DashboardView extends ConsumerWidget {
  const DashboardView({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final dashboard = ref.watch(dashboardProvider);

    return dashboard.when(
      loading: () => const DashboardSkeleton(),
      error: (error, stackTrace) => DashboardError(
        message: '기기 상태를 불러오지 못했다.',
        onRetry: () => ref.invalidate(dashboardProvider),
      ),
      data: (state) => DeviceDashboard(state: state),
    );
  }
}
```

처음엔 `error.toString()`을 그대로 보여줬다. 해보니 별로였다. `CBErrorPeripheralDisconnected`나 MQTT socket 예외가 그대로 노출되면 사용자는 할 수 있는 게 없다. 화면에는 짧은 문장을 보여주고, 자세한 에러는 ProviderObserver나 Crashlytics로 넘기는 편이 낫다.

재시도 버튼도 Controller 메서드로 빼지 않았다. 화면이 다시 읽어야 하는 Provider를 명확히 알고 있다면 `ref.invalidate(dashboardProvider)`가 더 단순하다. 단, BLE 연결 자체를 끊고 다시 붙이는 동작까지 여기 넣으면 안 된다. 화면 새로고침과 실제 통신 세션 재연결은 레벨이 다르다. [BLE 재연결 전략]({% post_url 2026-05-11-09-24-36-148140-ble-connection-stability-reconnect-error-handling %})에서 따로 뺐던 이유도 그거였다.

주의할 점은 `AsyncValue`가 모든 상태 설계를 대신해주지는 않는다는 것이다. 보일러 전원 토글처럼 사용자가 누른 직후 낙관적 UI가 필요한 값은 별도 command 상태가 필요하다. 읽기 상태는 `AsyncValue`, 쓰기 명령은 `Notifier`나 별도 pending 플래그로 분리하는 쪽이 덜 헷갈렸다.

짧게 남기면 이렇다.

- Flutter IoT 대시보드의 읽기 상태는 `AsyncValue`로 통일하면 화면 분기가 줄어든다.
- Flutter BLE와 MQTT가 잠깐 흔들려도 기존 데이터를 전부 지우지 않는 설계가 실사용에서 더 안정적이다.
- `ref.invalidate`는 화면 데이터 재조회에 쓰고, 실제 BLE 재연결 같은 세션 제어와 섞지 않는다.
