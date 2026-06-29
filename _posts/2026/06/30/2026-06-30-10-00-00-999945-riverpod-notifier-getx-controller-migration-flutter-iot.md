---
layout: post
title: "Riverpod Notifier로 GetX Controller 교체 — IoT 앱 마이그레이션 1단계"
description: "GetX 레포 소멸 이후 IoT 스마트홈 앱을 Riverpod 3.0으로 전환하는 첫 단계 실전 코드. BleController의 Rx 변수를 NotifierProvider로, Obx 위젯을 ref.watch로 교체하는 과정을 실제 코드로 정리했다."
date: 2026-06-30
tags: [Flutter, Riverpod, GetX, 상태관리, IoT]
comments: true
share: true
---

![Flutter 상태관리 마이그레이션 코드 리팩토링](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

[지난 글]({% post_url 2026-06-29-18-00-00-987600-getx-deprecated-riverpod-migration-flutter-iot %})에서 GetX 레포 소멸 상황을 정리하고 Riverpod 3.0 전환을 결정했다. 이번엔 실제로 손을 댄 첫 번째 단계 — GetX Controller를 Riverpod Notifier로 교체하는 과정이다. 전체 앱을 한 번에 바꾸면 망한다. Controller 하나씩, 기능 하나씩 갈아타는 게 현실적으로 돌아가는 방법이다.

## 공존 설정부터 — GetX와 Riverpod이 같은 앱에 있어도 된다

처음엔 두 라이브러리가 충돌할 줄 알았다. 근데 `ProviderScope`와 `GetMaterialApp`은 같이 쓸 수 있다. 마이그레이션 기간 동안 GetX가 아직 살아있는 화면들은 그대로 두면 된다.

```yaml
# pubspec.yaml
dependencies:
  get: ^4.6.6          # 기존 — 아직 살려둠
  flutter_riverpod: ^3.0.0
  riverpod_annotation: ^3.0.0

dev_dependencies:
  riverpod_generator: ^3.0.0
  build_runner: ^2.4.0
```

```dart
// main.dart
void main() {
  runApp(
    ProviderScope(          // Riverpod 루트
      child: MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(  // GetX 라우팅은 그대로
      home: HomeScreen(),
    );
  }
}
```

`ProviderScope`가 바깥이고 `GetMaterialApp`이 안쪽이다. 이 순서가 바뀌면 Riverpod ref를 못 찾아서 에러가 난다.

## GetX Controller vs Riverpod Notifier — 뭐가 달라지나

실제 `BleController` 기준으로 비교하면 이렇다.

**GetX 버전:**
```dart
class BleController extends GetxController {
  final _scanResults = <ScanResult>[].obs;
  final _isConnected = false.obs;
  final _connectedDevice = Rxn<BluetoothDevice>();

  List<ScanResult> get scanResults => _scanResults;
  bool get isConnected => _isConnected.value;
  BluetoothDevice? get connectedDevice => _connectedDevice.value;

  Future<void> startScan() async {
    _scanResults.clear();
    await FlutterBluePlus.startScan(timeout: Duration(seconds: 10));
    FlutterBluePlus.scanResults.listen((results) {
      _scanResults.assignAll(results);
    });
  }

  Future<void> connect(BluetoothDevice device) async {
    await device.connect();
    _isConnected.value = true;
    _connectedDevice.value = device;
  }
}
```

**Riverpod 3.0 Notifier 버전:**
```dart
// ble_state.dart
class BleState {
  const BleState({
    this.scanResults = const [],
    this.isConnected = false,
    this.connectedDevice,
  });

  final List<ScanResult> scanResults;
  final bool isConnected;
  final BluetoothDevice? connectedDevice;

  BleState copyWith({
    List<ScanResult>? scanResults,
    bool? isConnected,
    BluetoothDevice? connectedDevice,
  }) {
    return BleState(
      scanResults: scanResults ?? this.scanResults,
      isConnected: isConnected ?? this.isConnected,
      connectedDevice: connectedDevice ?? this.connectedDevice,
    );
  }
}

// ble_notifier.dart
@riverpod
class BleNotifier extends _$BleNotifier {
  @override
  BleState build() => const BleState();

  Future<void> startScan() async {
    state = state.copyWith(scanResults: []);
    await FlutterBluePlus.startScan(timeout: const Duration(seconds: 10));
    FlutterBluePlus.scanResults.listen((results) {
      state = state.copyWith(scanResults: results);
    });
  }

  Future<void> connect(BluetoothDevice device) async {
    await device.connect();
    state = state.copyWith(
      isConnected: true,
      connectedDevice: device,
    );
  }
}
```

변화의 핵심은 두 가지다. `.obs` Rx 변수들이 단일 `BleState` immutable 객체로 합쳐지고, `GetxController` 대신 `_$BleNotifier`를 extends한다. 코드 생성기가 `_$BleNotifier` 기반 클래스를 만들어주기 때문에 `build_runner` 실행이 필수다.

```bash
dart run build_runner build --delete-conflicting-outputs
```

## Obx를 ref.watch로 교체

위젯 쪽이 오히려 더 단순해진다.

**GetX 버전:**
```dart
class BleScreen extends StatelessWidget {
  final controller = Get.find<BleController>();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Obx(() => ListView.builder(
        itemCount: controller.scanResults.length,
        itemBuilder: (ctx, i) => ListTile(
          title: Text(controller.scanResults[i].device.remoteId.str),
        ),
      )),
    );
  }
}
```

**Riverpod 버전:**
```dart
class BleScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final bleState = ref.watch(bleNotifierProvider);

    return Scaffold(
      body: ListView.builder(
        itemCount: bleState.scanResults.length,
        itemBuilder: (ctx, i) => ListTile(
          title: Text(bleState.scanResults[i].device.remoteId.str),
        ),
      ),
    );
  }
}
```

`StatelessWidget` → `ConsumerWidget`, `Obx(()=>...)` → `ref.watch(...)`. `Get.find<BleController>()` 라인도 사라진다. 전체적으로 보일러플레이트가 줄었다.

메서드 호출할 때는 `ref.read`를 쓴다:

```dart
ElevatedButton(
  onPressed: () => ref.read(bleNotifierProvider.notifier).startScan(),
  child: Text('스캔 시작'),
)
```

`ref.watch`는 상태를 구독하고, `ref.read`는 한 번 읽거나 메서드를 호출할 때 쓴다. GetX에서 `controller.method()` 쓰던 패턴이 `ref.read(provider.notifier).method()`로 바뀌는 것뿐이다.

## 삽질했던 부분

**build_runner 안 돌리면 컴파일 에러 폭탄이다.** `@riverpod` 어노테이션 달고 저장하면 바로 빨간 줄이 생긴다. `_$BleNotifier` 클래스가 생성 안 됐기 때문이다. 처음엔 뭔가 잘못 설치된 줄 알고 30분을 날렸다.

```bash
# watch 모드로 켜두면 편하다
dart run build_runner watch --delete-conflicting-outputs
```

**`GetMaterialApp` 안에 `ProviderScope` 넣으면 안 된다.** 반드시 바깥이어야 한다. 안에 넣으면 Widget tree 상위에 `ProviderScope`가 없다는 에러가 뜬다:

```
ProviderNotFoundException: No ProviderScope found in the widget tree
```

**Rx 변수를 일대일로 NotifierProvider로 만들지 않는다.** 처음에 `.obs` 변수 3개를 Provider 3개로 나눠서 만들었다가 상태 동기화가 지옥이 됐다. 연관된 상태는 하나의 State 클래스로 묶는 게 맞다. `BleState`처럼.

**Stream 구독 관리.** GetX는 `onClose()`에서 subscription.cancel()을 했는데, Riverpod은 `ref.onDispose()`를 쓴다:

```dart
@riverpod
class BleNotifier extends _$BleNotifier {
  StreamSubscription? _scanSub;

  @override
  BleState build() {
    ref.onDispose(() => _scanSub?.cancel());
    return const BleState();
  }

  Future<void> startScan() async {
    state = state.copyWith(scanResults: []);
    _scanSub = FlutterBluePlus.scanResults.listen((results) {
      state = state.copyWith(scanResults: results);
    });
    await FlutterBluePlus.startScan(timeout: const Duration(seconds: 10));
  }
}
```

`ref.onDispose()` 안에서 구독 취소 처리를 해야 메모리 누수가 없다.

![Riverpod Provider 구조 다이어그램](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## 현재 진행 상황

BleController 하나를 Notifier로 교체했다. 앱이 여전히 돌아간다. GetX 라우팅과 나머지 Controller들은 손대지 않았다.

점진적 교체의 핵심은 이거다 — 한 Controller 교체 → 빌드 확인 → 테스트 → 다음 Controller. 전부 바꾸고 한 번에 빌드하면 에러가 어디서 났는지 추적이 안 된다.

다음 편은 `MqttController`를 Notifier로 바꾸는 과정이다. BLE보다 복잡한 게, MQTT는 연결 상태·토픽 구독·메시지 스트림이 얽혀 있어서 State 설계부터 다시 생각해야 한다.

**참고 링크**
- [Riverpod 3.0 공식 마이그레이션 가이드](https://riverpod.dev/docs/3.0_migration)
- [flutter_riverpod pub.dev](https://pub.dev/packages/flutter_riverpod)
- [riverpod_annotation pub.dev](https://pub.dev/packages/riverpod_annotation)
