---
layout: post
title: "Flutter IoT Riverpod BLE permission state - Flutter BLE 스캔 전 권한 상태 분리하기"
description: "Flutter IoT 앱에서 Riverpod으로 Flutter BLE 권한 상태와 스캔 Provider를 분리해 권한 거부, 재요청, 설정 이동 흐름이 꼬이지 않게 정리했다."
date: 2026-07-04
tags: [Flutter, Riverpod, BLE, flutter_blue_plus, permission_handler, IoT, 스마트홈]
comments: true
share: true
---

![Flutter IoT Riverpod BLE permission state](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
이 그림에서는 사용자가 버튼을 누른 뒤 권한 확인, BLE 스캔, 기기 목록 표시가 서로 다른 단계로 움직이는 지점을 보면 된다.

Flutter IoT 앱에서 Riverpod으로 Flutter BLE 스캔을 만들 때 권한 상태는 `flutter_blue_plus` 스캔 Stream과 분리하는 편이 낫다. Flutter 스마트홈 기기 등록 화면에서 권한 거부와 스캔 실패를 같은 에러로 묶으면, 사용자는 설정으로 가야 하는지 다시 스캔 버튼을 눌러야 하는지 알 수 없다.

처음엔 `scanProvider` 안에서 `permission_handler`를 호출했다. 버튼을 누르면 권한을 확인하고, 허용이면 바로 BLE 스캔을 시작하는 구조였다. 작은 예제에서는 깔끔해 보였다. 해보니 실제 앱에서는 애매했다. Android 12 이상은 `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`가 따로 있고, Android 11 이하는 위치 권한이 끼어든다. iOS는 실기기에서만 의미 있는 BLE 권한 타이밍이 있다. 이걸 스캔 Stream 안에 넣으니 테스트도 UI도 같이 흔들렸다.

| 상태 | 화면 행동 | Provider 책임 |
|---|---|---|
| `unknown` | 버튼 비활성 또는 안내 유지 | 아직 확인 전 |
| `granted` | 스캔 시작 가능 | 스캔 Provider 구독 허용 |
| `denied` | 재요청 버튼 표시 | 권한 요청 이벤트 대기 |
| `permanentlyDenied` | 설정 이동 버튼 표시 | 앱 설정 열기 |
| `unsupported` | BLE 불가 안내 | 기기/OS 조건 표시 |

권한 Provider는 값만 가진다. 스캔을 시작하지 않는다.

```dart
enum BlePermissionState {
  unknown,
  granted,
  denied,
  permanentlyDenied,
  unsupported,
}

final blePermissionProvider =
    NotifierProvider<BlePermissionNotifier, BlePermissionState>(
  BlePermissionNotifier.new,
);

class BlePermissionNotifier extends Notifier<BlePermissionState> {
  @override
  BlePermissionState build() => BlePermissionState.unknown;

  Future<void> check() async {
    final bluetoothScan = await Permission.bluetoothScan.status;
    final bluetoothConnect = await Permission.bluetoothConnect.status;

    if (bluetoothScan.isGranted && bluetoothConnect.isGranted) {
      state = BlePermissionState.granted;
      return;
    }

    if (bluetoothScan.isPermanentlyDenied ||
        bluetoothConnect.isPermanentlyDenied) {
      state = BlePermissionState.permanentlyDenied;
      return;
    }

    state = BlePermissionState.denied;
  }
}
```

스캔 Provider는 권한이 허용된 뒤에만 구독한다. 여기서 권한 요청을 다시 하지 않는 게 핵심이다.

```dart
final bleScanProvider = StreamProvider.autoDispose<List<BleDeviceSummary>>((ref) {
  final permission = ref.watch(blePermissionProvider);

  if (permission != BlePermissionState.granted) {
    return const Stream.empty();
  }

  final bleRepository = ref.watch(bleRepositoryProvider);
  return bleRepository.scanDevices(timeout: const Duration(seconds: 10));
});
```

여기서 헷갈렸던 건 `Stream.empty()`가 실패를 숨기는 것처럼 보인다는 점이다. 그런데 실패가 아니다. 권한이 없으면 아직 스캔 조건이 안 된 것이다. 화면은 `blePermissionProvider`를 보고 "권한 허용 필요"를 보여주고, 기기 목록 영역은 빈 상태로 둔다. 반대로 권한은 있는데 스캔 결과가 없으면 "주변 기기를 찾지 못함"이라고 말한다. 이 둘을 나누니 고객센터 로그도 훨씬 읽기 쉬웠다.

[BLE 재연결 안정성 문제]({% post_url 2026-05-11-09-24-36-148140-ble-connection-stability-reconnect-error-handling %})에서도 비슷한 판단을 했다. 연결 실패, 권한 실패, 타임아웃을 한 덩어리로 보면 원인을 못 찾는다. Riverpod 전환 후에는 상태가 더 잘게 보이니, 오히려 에러 메시지를 덜 과장해서 보여줄 수 있었다.

주의할 점은 권한 요청 자체를 Provider `build()` 안에서 하면 안 된다는 것이다. Provider가 재생성될 때 시스템 권한 팝업이 다시 뜰 수 있다. 나는 버튼 클릭이나 화면 진입 이벤트에서 `check()` 또는 `request()`를 호출하고, Provider는 결과 상태만 들고 있게 했다. 테스트에서도 이 구조가 편했다. `granted`를 주입하면 스캔 Provider만 검증하고, `permanentlyDenied`를 주입하면 설정 이동 버튼 노출만 보면 된다.

짧게 정리하면 이렇다.

- Flutter BLE 권한 상태와 스캔 Stream은 Riverpod Provider를 나눠서 관리한다.
- 권한 없음, 영구 거부, 스캔 결과 없음은 서로 다른 UI 문구로 보여준다.
- `permission_handler` 호출은 사용자 이벤트에 묶고, Provider `build()`에서는 팝업을 띄우지 않는다.
- IoT 기기 등록 화면에서는 이 분리가 디버깅 로그와 테스트 코드를 동시에 단순하게 만든다.
