---
layout: post
title: "Flutter BLE 스캔 상태 - Riverpod StreamProvider로 GetX Controller 줄이기"
description: "Flutter IoT 앱에서 Flutter BLE 스캔 결과를 Riverpod StreamProvider로 분리해 GetX Controller 비대화를 줄인 과정을 정리했다."
date: 2026-07-01
tags: [Flutter, BLE, flutter_blue_plus, GetX, IoT]
comments: true
share: true
---
![Flutter BLE 스캔 상태 관리](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

Flutter IoT 앱에서 Flutter BLE 스캔 상태는 화면 Controller에 오래 두면 금방 꼬인다. Flutter 스마트홈 기기 등록 화면에서 `flutter_blue_plus 사용법`만 보고 스캔을 붙였을 때는 동작했다. 문제는 스캔 중, 권한 거부, 기기 발견, 중복 제거, 자동 중지 상태가 전부 GetX Controller 안으로 들어오면서부터였다. Riverpod으로 옮길 때는 BLE 스캔 결과를 `StreamProvider`로 빼는 쪽이 훨씬 관리하기 쉬웠다.

[Flutter BLE 기초]({% post_url 2026-05-05-09-42-18-074070-ble-bluetooth-basics-flutter-blue-plus-scanning %})를 만들 때는 스캔 호출 자체가 핵심이었다. 실제 앱에서는 호출보다 상태 정리가 더 어렵다. iOS 실기기에서는 스캔 결과가 늦게 들어오고, Android에서는 같은 기기가 RSSI만 바뀐 채 여러 번 들어온다. 여기에 기기 등록 바텀시트가 닫혔는데 스캔이 계속 도는 실수도 있었다.

## GetX에 남기면 커지는 코드

[GetX 상태관리]({% post_url 2026-05-02-09-21-39-037035-getx-state-management-routing-flutter-iot %})로 처음 구현했을 때 구조는 단순했다. `BleScanController`가 권한 요청, 스캔 시작, 결과 목록, 로딩 플래그, 에러 메시지까지 들고 있었다.

처음엔 편했다. 버튼에서 `controller.startScan()`을 부르면 되고, 화면은 `Obx`로 리스트만 그리면 됐다. 그런데 MQTT 연결 상태, WiFi 프로비저닝, 기기 등록 진행률까지 같은 플로우에 붙으니 Controller가 화면인지 서비스인지 애매해졌다. 특히 스캔 결과는 화면 수명보다 BLE 세션 수명에 더 가깝다.

## StreamProvider로 스캔 결과를 읽는다

BLE Wrapper는 그대로 두고, Riverpod Provider가 Stream을 노출하게 만들었다. UI는 스캔 결과를 소유하지 않고 구독만 한다.

```dart
final bleRepositoryProvider = Provider<BleRepository>((ref) {
  return FlutterBlueBleRepository();
});

final bleScanProvider =
    StreamProvider.autoDispose<List<BleDeviceSummary>>((ref) {
  final repository = ref.watch(bleRepositoryProvider);

  ref.onDispose(() {
    repository.stopScan();
  });

  return repository.scanDevices(timeout: const Duration(seconds: 10));
});
```

핵심은 `autoDispose`다. 기기 등록 화면을 닫으면 Provider도 정리되고, `ref.onDispose`에서 스캔을 멈춘다. 예전에는 `onClose()`에 `stopScan()`을 넣었는데, 라우팅 중간에 바텀시트만 닫히는 경우를 놓친 적이 있다. Riverpod에서는 화면이 더 이상 구독하지 않는 상태를 정리 기준으로 삼을 수 있다.

## UI는 AsyncValue만 처리한다

화면 코드는 오히려 줄었다. 스캔 중인지, 실패했는지, 결과가 비었는지를 `AsyncValue` 분기로만 본다.

```dart
class BleScanView extends ConsumerWidget {
  const BleScanView({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final scan = ref.watch(bleScanProvider);

    return scan.when(
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (error, stackTrace) => BleScanErrorView(message: '$error'),
      data: (devices) {
        if (devices.isEmpty) {
          return const EmptyBleDeviceView();
        }

        return ListView.builder(
          itemCount: devices.length,
          itemBuilder: (context, index) {
            final device = devices[index];
            return BleDeviceTile(device: device);
          },
        );
      },
    );
  }
}
```

이 방식의 장점은 실패 지점이 분리된다는 데 있다. BLE 권한 문제는 Repository에서 예외로 올리고, 화면은 에러 UI만 보여준다. 중복 기기 제거도 Repository 내부에서 처리한다. UI는 `List<BleDeviceSummary>`만 받으니 테스트도 쉬워진다.

## 조심할 지점

스캔 시작 버튼을 누를 때마다 Provider를 새로 만들고 싶다면 `ref.invalidate(bleScanProvider)`를 써야 한다. 같은 Provider를 계속 watch만 하면 사용자가 재시도 버튼을 눌러도 이전 에러 상태가 남아 보일 수 있다.

또 하나는 권한 요청 위치다. 권한 다이얼로그는 화면 이벤트에 가깝고, BLE 스캔 Stream은 데이터 흐름에 가깝다. 둘을 한 Provider에 섞으면 테스트하기 애매해진다. 나는 버튼 클릭에서 권한을 확인하고, 허용된 뒤에만 스캔 Provider를 구독하는 구조로 나눴다.

짧게 정리하면:

- Flutter BLE 스캔 결과는 GetX Controller 필드보다 StreamProvider에 어울린다.
- `autoDispose`와 `ref.onDispose`를 같이 쓰면 화면 이탈 시 스캔 중지를 놓칠 가능성이 줄어든다.
- 권한 요청, 스캔 Stream, UI 상태를 분리하면 Flutter IoT 기기 등록 플로우가 덜 비대해진다.
