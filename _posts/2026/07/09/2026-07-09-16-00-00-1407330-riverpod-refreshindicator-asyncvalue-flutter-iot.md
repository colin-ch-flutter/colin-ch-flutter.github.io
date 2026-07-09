---
layout: post
title: "Flutter IoT Riverpod RefreshIndicator - 스마트홈 대시보드 새로고침 깜빡임 줄이기"
description: "Flutter IoT 앱에서 Riverpod AsyncValue와 RefreshIndicator로 Flutter 스마트홈 대시보드 새로고침 때 Flutter BLE·MQTT 카드가 깜빡이는 문제를 줄였다."
date: 2026-07-09
tags: [Flutter, Riverpod, IoT, 스마트홈, MQTT, BLE]
comments: true
share: true
---
![Flutter IoT Riverpod RefreshIndicator 스마트홈 대시보드](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

이 그림에서는 스마트홈 대시보드처럼 여러 기기 상태가 한 화면에 모일 때 새로고침 표시를 어디까지 보여줄지 봐야 한다.

Flutter IoT 앱에서 Riverpod `AsyncValue`와 `RefreshIndicator`를 같이 쓸 때 핵심은 초기 로딩과 사용자 새로고침을 분리하는 것이다. Flutter 스마트홈 대시보드에서 Flutter BLE 연결 카드와 MQTT 보일러 상태 카드를 매번 빈 화면으로 바꾸면 앱이 불안정해 보인다. 새 데이터는 가져오되, 직전 값은 화면에 남겨두는 쪽이 더 자연스러웠다.

처음엔 단순했다. 사용자가 아래로 당기면 `ref.invalidate(dashboardProvider(spaceId))`를 호출하고, `AsyncLoading`이면 전체 로딩 위젯을 보여줬다. 해보니 됐다. 그런데 실제 기기 6개가 붙은 화면에서는 문제가 바로 보였다. MQTT 응답은 300ms 안에 왔지만 BLE RSSI와 연결 상태는 1초 가까이 늦었다. 그 사이 카드 전체가 사라졌다가 다시 나타나니, 사용자는 보일러가 끊긴 줄 알았다.

## 초기 로딩과 새로고침은 다르다

내가 정한 기준은 이렇다. 앱을 처음 열었고 표시할 값이 없으면 로딩 화면을 보여준다. 이미 값이 있다면 새로고침 중에도 기존 카드를 유지하고, 상단에만 진행 상태를 둔다.

| 상황 | 화면 처리 | 이유 |
|---|---|---|
| 첫 진입, 캐시 없음 | 전체 로딩 | 보여줄 기준값이 없다 |
| pull-to-refresh | 기존 카드 유지 | 사용자는 최신화만 기대한다 |
| 일부 기기 실패 | 실패 카드만 표시 | 전체 대시보드를 죽이지 않는다 |
| Space 변경 | 이전 Space 값 제거 | 다른 집 상태가 섞이면 위험하다 |

대시보드 Provider는 통신 세션을 다시 만들지 않고, 현재 Space의 스냅샷만 다시 읽게 했다.

```dart
final dashboardProvider =
    FutureProvider.autoDispose.family<DashboardState, String>((ref, spaceId) async {
  final repository = ref.watch(dashboardRepositoryProvider);
  return repository.fetchSnapshot(spaceId);
});
```

화면에서는 `AsyncValue.when`을 그대로 쓰되, 새로고침 중 기존 값을 버리지 않는 쪽으로 조건을 잡았다.

```dart
class DashboardPage extends ConsumerWidget {
  const DashboardPage({super.key, required this.spaceId});

  final String spaceId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final dashboard = ref.watch(dashboardProvider(spaceId));
    final cached = dashboard.valueOrNull;

    return RefreshIndicator(
      onRefresh: () async {
        ref.invalidate(dashboardProvider(spaceId));
        await ref.read(dashboardProvider(spaceId).future);
      },
      child: dashboard.when(
        skipLoadingOnRefresh: true,
        data: (state) => DeviceListView(state: state),
        loading: () {
          if (cached != null) {
            return DeviceListView(state: cached, refreshing: true);
          }
          return const DashboardSkeleton();
        },
        error: (error, stackTrace) {
          if (cached != null) {
            return DeviceListView(
              state: cached,
              bannerText: '일부 상태를 새로 가져오지 못했다',
            );
          }
          return DashboardErrorView(error: error);
        },
      ),
    );
  }
}
```

여기서 헷갈렸던 건 `invalidate`가 MQTT 재연결 버튼처럼 보인다는 점이다. 하지만 대시보드 새로고침은 화면 데이터 조회다. MQTT 세션을 끊고 다시 붙이는 동작과 섞으면 publish 대기 중인 명령까지 흔들린다. [클린 아키텍처 구조]({% post_url 2026-05-01-11-14-26-024690-clean-architecture-large-flutter-project-structure %})를 지키려면 화면은 조회 Provider만 갱신하고, BLE 재연결이나 MQTT reconnect 정책은 Repository 아래에 둬야 한다.

## 실패를 한 덩어리로 보지 않는다

스마트홈 대시보드는 실패 원인이 섞인다. 보일러 MQTT 상태는 정상인데 BLE 온습도 센서만 멀리 있어서 늦을 수 있다. 반대로 BLE는 잡히는데 AWS IoT 토픽 구독이 늦는 경우도 있었다. 그래서 `DashboardState` 안에 기기별 상태를 나눴다.

```dart
class DeviceCardState {
  const DeviceCardState({
    required this.deviceId,
    required this.powerText,
    required this.connectionText,
    this.warningText,
  });

  final String deviceId;
  final String powerText;
  final String connectionText;
  final String? warningText;
}
```

이렇게 두면 새로고침 실패가 전체 에러 화면으로 번지지 않는다. 보일러 카드는 마지막 MQTT 값을 유지하고, BLE 센서 카드만 `최근 수신 38초 전` 같은 경고를 붙일 수 있다. 처음엔 실패를 하나의 `Exception`으로 올렸는데, 실제 운영 로그를 보니 원인이 2개 이상 동시에 나오는 날이 있었다. 하나로 뭉개면 디버깅도 느려졌다.

짧게 남기면 이렇다. Flutter IoT 대시보드에서 `RefreshIndicator`는 데이터를 다시 읽는 신호일 뿐, 통신 세션을 재시작하는 버튼이 아니다. Riverpod `AsyncValue`는 첫 로딩과 새로고침을 나눠 처리하고, 직전 값이 있으면 화면을 유지한다. 그래야 Flutter 스마트홈 앱에서 BLE와 MQTT 응답 속도 차이가 카드 깜빡임으로 보이지 않는다.
