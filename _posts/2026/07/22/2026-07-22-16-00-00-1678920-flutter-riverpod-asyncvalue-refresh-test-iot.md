---
layout: post
title: "Flutter Riverpod AsyncValue 새로고침 테스트 - IoT 카드 깜빡임 막기"
description: "Flutter Riverpod AsyncValue의 isRefreshing 상태를 테스트해 MQTT·BLE 데이터 새로고침 중 기존 IoT 카드 값을 유지하고, 로딩과 에러 전이를 안전하게 검증하는 방법을 정리했다."
date: 2026-07-22
tags: [Flutter, Dart, Riverpod, MQTT, BLE, 테스트, IoT, 스마트홈]
comments: true
share: true
---

![Flutter Riverpod AsyncValue 새로고침 테스트와 IoT 스마트홈 카드](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

Flutter Riverpod `AsyncValue` 새로고침 테스트의 핵심은 `loading`만 확인하지 않고 `isRefreshing`일 때 기존 값을 계속 보여주는지 검증하는 데 있다. Flutter 스마트홈 대시보드에서 사용자가 당겨서 새로고침을 실행하면 MQTT 응답은 300ms 안에 오지만 BLE 연결 정보는 1초 정도 늦게 들어오는 경우가 있었다. 이때 화면 전체를 빈 로딩 상태로 바꾸면 보일러가 꺼진 것처럼 보인다.

## 새로고침과 첫 로딩은 다르다

처음엔 `state = const AsyncLoading()`만 넣고 테스트했다. 해보니 첫 진입과 새로고침의 사용자 경험이 완전히 달랐다. 첫 진입에는 스켈레톤이 맞지만, 이미 값이 있는 상태에서는 이전 값을 유지하면서 작은 진행 표시만 보여야 한다.

| 상태 | 카드에 보여줄 값 | 테스트 기준 |
|---|---|---|
| 첫 진입 | 없음 또는 스켈레톤 | `isLoading == true` |
| 새로고침 중 | 직전 MQTT·BLE 값 | `isRefreshing == true` |
| 새로고침 성공 | 최신 값 | `hasValue == true` |
| 새로고침 실패 | 직전 값 + 오류 표시 | `hasValue && hasError` |

`isRefreshing`은 데이터가 이미 있는데 다시 로딩 중인 상황을 표현한다. 이 플래그를 UI에서 쓰기로 했다면 단순히 최종 값만 검사하는 테스트는 부족하다.

## Fake Repository로 새로고침 타이밍을 고정한다

실제 네트워크 대신 `Completer`로 응답 시점을 직접 조절하면 상태 전이를 흔들림 없이 확인할 수 있다.

```dart
class FakeDashboardRepository implements DashboardRepository {
  Completer<DashboardState>? pending;

  @override
  Future<DashboardState> fetch(String spaceId) {
    final request = Completer<DashboardState>();
    pending = request;
    return request.future;
  }
}

class DashboardNotifier extends AutoDisposeAsyncNotifier<DashboardState> {
  @override
  Future<DashboardState> build() =>
      ref.read(dashboardRepositoryProvider).fetch('home');

  Future<void> refresh() async {
    final repository = ref.read(dashboardRepositoryProvider);
    final previous = state;
    state = const AsyncLoading<DashboardState>().copyWithPrevious(previous);
    final result = await AsyncValue.guard(() => repository.fetch('home'));
    state = result.hasError
        ? AsyncError<DashboardState>(result.error!, result.stackTrace!)
            .copyWithPrevious(previous)
        : result;
  }
}

final dashboardProvider =
    AsyncNotifierProvider.autoDispose<DashboardNotifier, DashboardState>(
  DashboardNotifier.new,
);
```

여기서 `copyWithPrevious(state)`를 빼먹으면 새로고침 순간 값이 사라진다. `AsyncValue.guard`는 예외를 `AsyncError`로 바꿔주지만, 이전 값 보존 여부는 자동으로 결정해주지 않는다. 이 부분은 Provider 구현의 책임으로 남는다.

## isRefreshing과 실패 상태를 같이 검증한다

초기 요청을 성공시킨 뒤, 두 번째 요청이 끝나기 전과 실패한 뒤를 각각 확인한다.

```dart
test('새로고침 중 기존 값을 유지하고 실패를 표시한다', () async {
  final repository = FakeDashboardRepository();
  final container = ProviderContainer(overrides: [
    dashboardRepositoryProvider.overrideWithValue(repository),
  ]);
  addTearDown(container.dispose);

  final listener = container.listen(dashboardProvider, (_, __) {});
  repository.pending!.complete(const DashboardState(temperature: 22));
  await container.read(dashboardProvider.future);

  final refresh = container.read(dashboardProvider.notifier).refresh();
  final refreshing = container.read(dashboardProvider);
  expect(refreshing.isRefreshing, isTrue);
  expect(refreshing.requireValue.temperature, 22);

  repository.pending!.completeError(StateError('MQTT timeout'));
  await refresh;

  final failed = container.read(dashboardProvider);
  expect(failed.hasError, isTrue);
  expect(failed.requireValue.temperature, 22);
  listener.close();
});
```

처음에는 `container.read(dashboardProvider.future)`만 기다리면 모든 검증이 끝날 줄 알았다. 하지만 새로고침 Future를 별도 변수로 잡지 않으면 응답 전 상태를 검사할 수 없다. `refresh()`를 호출한 즉시 상태를 읽고, Fake 응답을 완료한 뒤 최종 상태를 읽는 순서가 포인트다.

## UI에서는 작은 갱신 표시만 바꾼다

화면은 `hasValue`를 우선 처리하고 `isRefreshing`을 보조 상태로 사용한다. 오류가 났더라도 직전 값이 있으면 전체 에러 화면 대신 카드와 재시도 버튼을 함께 보여주는 편이 IoT 앱에 맞았다.

```dart
final state = ref.watch(dashboardProvider);

return state.when(
  loading: () => const DashboardSkeleton(),
  error: (error, _) => ErrorCard(error: error),
  data: (data) => DashboardCard(
    data: data,
    refreshing: state.isRefreshing,
    hasRefreshError: state.hasError,
  ),
);
```

다만 `when`의 `error` 분기는 이전 값이 보존된 `AsyncError`를 별도 카드로 보여주지 않는다. 실제 UI에서는 `state.hasValue`를 확인한 뒤 분기하거나, `skipError: true` 같은 옵션을 화면 의도에 맞게 선택해야 한다. 이 선택까지 Widget Test에 넣어야 테스트가 구현 세부사항이 아니라 사용자 경험을 보호한다.

짧게 남기면 이렇다. Flutter Riverpod 새로고침은 첫 로딩과 같은 `loading`으로 취급하면 안 된다. `copyWithPrevious`로 기존 값을 보존하고, `isRefreshing`과 새 요청의 성공·실패를 각각 테스트해야 MQTT와 BLE 응답 속도가 달라도 스마트홈 카드가 깜빡이지 않는다.
