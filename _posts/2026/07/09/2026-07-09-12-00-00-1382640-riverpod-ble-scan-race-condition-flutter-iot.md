---
layout: post
title: "Flutter BLE Riverpod Race Condition - 늦게 도착한 스캔 결과 버리기"
description: "Flutter IoT 앱에서 Riverpod과 flutter_blue_plus로 BLE 스캔을 처리할 때 늦게 도착한 결과가 최신 화면을 덮는 문제와 방어 기준을 정리했다."
date: 2026-07-09
tags: [Flutter, Riverpod, BLE, flutter_blue_plus, IoT]
comments: true
share: true
---

![Flutter BLE Riverpod 스캔 경쟁 상태](https://images.unsplash.com/photo-1558346490-a72e53ae2d4f?w=800&q=80)
이 그림에서는 무선 기기 검색처럼 결과가 늦게 도착할 수 있는 흐름을 봐야 한다.

Flutter IoT 앱에서 Flutter BLE 스캔은 `startScan()`보다 "이 결과가 아직 유효한가"를 더 신경 써야 했다. Riverpod으로 상태를 나눠도 flutter_blue_plus 이벤트가 늦게 도착하면 이전 화면의 스캔 결과가 최신 화면을 덮는다. 내가 확인한 기준으론 스캔 요청마다 번호를 붙이고, 현재 요청 번호와 다르면 버리는 방식이 가장 단순하게 버텼다.

처음엔 `autoDispose`만 붙이면 해결될 줄 알았다. 등록 화면을 나가면 Provider가 정리되고, `ref.onDispose`에서 스캔도 멈추니까 충분해 보였다. 그런데 실제 기기에서 검색어를 바꾸거나 Space를 전환할 때 문제가 남았다. 화면은 300ms 안에 바뀌었는데 BLE advertisement는 1초 뒤에도 들어왔다. iOS 실기기에서는 특히 주변 기기가 많을수록 같은 이름의 결과가 뒤늦게 섞였다.

## 문제를 상태 기준으로 나눈다

| 상황 | 위험 | 처리 기준 |
|---|---|---|
| 화면 종료 | 스캔 계속 실행 | `ref.onDispose`에서 stop |
| 검색어 변경 | 이전 필터 결과 표시 | 요청 번호 비교 |
| Space 변경 | 다른 집 기기 섞임 | `family(spaceId)`로 분리 |
| 권한 거부 | 빈 목록처럼 보임 | 에러 상태 분리 |

여기서 헷갈렸던 건 `StreamSubscription.cancel()`이 이미 들어온 이벤트까지 없애준다고 착각한 부분이다. 구독을 끊어도 Repository 내부 큐나 플랫폼 채널에서 올라온 값이 한 박자 늦게 처리될 수 있다. 그래서 dispose와 별개로 "현재 스캔인지"를 확인하는 방어선이 필요했다.

스캔 요청 번호를 상태 안에 두고, 늦은 결과를 버리는 최소 형태는 이렇게 잡았다.

```dart
final bleScanQueryProvider =
    StateProvider.autoDispose<String>((ref) => '');

final bleScanProvider = StreamProvider.autoDispose
    .family<List<BleDeviceSummary>, String>((ref, spaceId) {
  final query = ref.watch(bleScanQueryProvider);
  final repository = ref.watch(bleRepositoryProvider);
  final requestId = DateTime.now().microsecondsSinceEpoch;

  var alive = true;
  ref.onDispose(() {
    alive = false;
    repository.stopScan(spaceId);
  });

  return repository
      .scan(spaceId: spaceId, query: query, requestId: requestId)
      .where((result) {
    if (!alive) return false;
    return result.requestId == requestId;
  }).map((result) => result.devices);
});
```

실제 코드에서는 `DateTime` 대신 Repository에서 증가시키는 `int scanSeq`를 썼다. 테스트에서 시간을 고정하기 쉽고, 로그도 `scanSeq=42 ignored`처럼 읽힌다. 핵심은 `spaceId`, `query`, `requestId`가 같은 흐름 안에서 만들어져야 한다는 점이다. 화면 위젯에서 임의로 필터링하면 Provider 상태와 로그가 어긋난다.

솔직하게 정리해보면 `autoDispose`는 청소 담당이고, race condition 방어는 검문 담당이다. 둘 중 하나만 있으면 부족했다. 특히 BLE는 실제 기기, OS 권한, 주변 전파 상태에 따라 이벤트 순서가 흔들린다. iOS 시뮬레이터에서는 BLE를 확인할 수 없으니 이 문제는 실기기에서만 보였다.

짧게 남기면 이렇다. Flutter BLE 스캔 Provider는 화면 수명, 검색 조건, 요청 번호를 따로 봐야 한다. Riverpod `family`로 Space를 나누고, `autoDispose`로 정리하고, 늦게 도착한 flutter_blue_plus 결과는 request id로 버린다. 그래야 Flutter 스마트홈 등록 화면에서 이전 집의 기기가 갑자기 나타나는 일을 줄일 수 있다.
