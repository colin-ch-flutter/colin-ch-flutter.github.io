---
layout: post
title: "flutter_blue_plus Unit Test - static 메서드를 Wrapper로 격리하는 법"
description: "FlutterBluePlus 1.10.0 이후 static 메서드만 있어 직접 Mock이 불가능하다. Wrapper 인터페이스 패턴으로 의존성을 격리하고 BLE 스캔·연결 로직을 Unit Test하는 방법을 실제 코드로 정리했다."
date: 2026-06-24
tags: [Flutter, UnitTest, BLE, flutter_blue_plus, Dart]
comments: true
share: true
---
![코드와 블루투스 연결](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

[이전 편에서 GetX Controller의 비동기 다이얼로그 테스트를 콜백 주입으로 해결했다.]({% post_url 2026-06-23-11-07-13-641940-flutter-getx-test-async-dialog-confirm %}) 이번엔 다른 종류의 벽을 만났다. `flutter_blue_plus`의 `FlutterBluePlus` 클래스가 static 메서드만 있어서, Mockito로 Mock 객체를 만들 수가 없다.

## 왜 flutter_blue_plus는 Mock이 안 되는가

`flutter_blue_plus` 1.10.0부터 싱글톤 인스턴스가 사라졌다. 이전에는 `FlutterBluePlus.instance`가 있었고 이걸 주입하면 됐다. 지금은 전부 static이다.

```dart
// 현재 API — 전부 static
FlutterBluePlus.startScan(timeout: const Duration(seconds: 5));
FlutterBluePlus.scanResults;  // Stream<List<ScanResult>>
FlutterBluePlus.stopScan();
```

Mockito의 `@GenerateMocks([FlutterBluePlus])`를 시도하면 빌드 자체가 안 된다. 클래스에 인스턴스 메서드가 없으니 Mock 생성 대상이 아니다.

처음엔 `flutter_blue_plus`가 제공하는 [Mocking Guide](https://pub.dev/packages/flutter_blue_plus#mocking)를 봤다. 거기서도 결론은 Wrapper를 만들라고 되어 있다.

## Wrapper 패턴 — abstract class로 인터페이스 만들기

핵심은 `FlutterBluePlus`의 static 호출을 감싸는 인스턴스 클래스를 하나 두는 것이다. Controller나 UseCase는 이 Wrapper만 바라본다.

```dart
// ble_service.dart
abstract class BleService {
  Stream<List<ScanResult>> get scanResults;
  Future<void> startScan({Duration timeout});
  Future<void> stopScan();
  Future<void> connect(BluetoothDevice device);
  Future<void> disconnect(BluetoothDevice device);
  Stream<BluetoothConnectionState> connectionState(BluetoothDevice device);
}
```

실제 구현체는 static 메서드를 그대로 위임한다.

```dart
// ble_service_impl.dart
class BleServiceImpl implements BleService {
  @override
  Stream<List<ScanResult>> get scanResults => FlutterBluePlus.scanResults;

  @override
  Future<void> startScan({Duration timeout = const Duration(seconds: 5)}) =>
      FlutterBluePlus.startScan(timeout: timeout);

  @override
  Future<void> stopScan() => FlutterBluePlus.stopScan();

  @override
  Future<void> connect(BluetoothDevice device) =>
      device.connect(timeout: const Duration(seconds: 10));

  @override
  Future<void> disconnect(BluetoothDevice device) => device.disconnect();

  @override
  Stream<BluetoothConnectionState> connectionState(BluetoothDevice device) =>
      device.connectionState;
}
```

실기기에서는 `BleServiceImpl`을 주입하고, 테스트에서는 Fake를 넣으면 된다.

## Controller는 BleService만 바라본다

```dart
class BleDeviceController extends GetxController {
  BleDeviceController({required BleService bleService})
      : _ble = bleService;

  final BleService _ble;
  final scanResults = <ScanResult>[].obs;
  final isScanning = false.obs;
  final connectionState = Rx<BluetoothConnectionState>(
    BluetoothConnectionState.disconnected,
  );

  Future<void> startScan() async {
    isScanning.value = true;
    scanResults.clear();

    _ble.scanResults.listen((results) {
      scanResults.assignAll(results);
    });

    await _ble.startScan(timeout: const Duration(seconds: 5));
    isScanning.value = false;
  }

  Future<void> connectDevice(BluetoothDevice device) async {
    await _ble.connect(device);
    _ble.connectionState(device).listen((state) {
      connectionState.value = state;
    });
  }
}
```

`FlutterBluePlus`를 직접 쓰던 코드랑 거의 다르지 않다. 인터페이스 하나 끼워넣었을 뿐이다.

## Fake 구현체

```dart
// fake_ble_service.dart
class FakeBleService implements BleService {
  final _scanResultsController =
      StreamController<List<ScanResult>>.broadcast();
  final _connectionControllers =
      <String, StreamController<BluetoothConnectionState>>{};

  List<ScanResult> fakeDevices = [];
  int startScanCallCount = 0;
  int stopScanCallCount = 0;

  @override
  Stream<List<ScanResult>> get scanResults =>
      _scanResultsController.stream;

  @override
  Future<void> startScan({Duration timeout = const Duration(seconds: 5)}) async {
    startScanCallCount++;
    // 바로 fakeDevices를 stream에 흘려보낸다
    _scanResultsController.add(fakeDevices);
  }

  @override
  Future<void> stopScan() async {
    stopScanCallCount++;
  }

  @override
  Future<void> connect(BluetoothDevice device) async {
    final controller = _getConnectionController(device.remoteId.str);
    controller.add(BluetoothConnectionState.connected);
  }

  @override
  Future<void> disconnect(BluetoothDevice device) async {
    final controller = _getConnectionController(device.remoteId.str);
    controller.add(BluetoothConnectionState.disconnected);
  }

  @override
  Stream<BluetoothConnectionState> connectionState(BluetoothDevice device) =>
      _getConnectionController(device.remoteId.str).stream;

  StreamController<BluetoothConnectionState> _getConnectionController(
      String deviceId) {
    return _connectionControllers.putIfAbsent(
      deviceId,
      () => StreamController<BluetoothConnectionState>.broadcast(),
    );
  }

  void dispose() {
    _scanResultsController.close();
    for (final c in _connectionControllers.values) {
      c.close();
    }
  }
}
```

`fakeDevices`에 원하는 `ScanResult` 목록을 세팅해두면 `startScan()` 호출 시 즉시 stream으로 흘러나온다. 실기기 없이도 테스트 가능하다.

## 테스트 코드

```dart
void main() {
  late FakeBleService fakeBle;
  late BleDeviceController controller;

  setUp(() {
    fakeBle = FakeBleService();
    controller = Get.put(BleDeviceController(bleService: fakeBle));
  });

  tearDown(() {
    fakeBle.dispose();
    Get.reset();
  });

  group('startScan', () {
    test('스캔 시작 시 isScanning이 true가 됐다가 false로 끝난다', () async {
      fakeBle.fakeDevices = [];

      await controller.startScan();

      expect(fakeBle.startScanCallCount, 1);
      expect(controller.isScanning.value, false);
    });

    test('fake 기기가 2개면 scanResults에 2개 들어온다', () async {
      fakeBle.fakeDevices = _makeFakeScanResults(['AA:BB:CC', 'DD:EE:FF']);

      await controller.startScan();
      await Future.delayed(Duration.zero); // stream flush

      expect(controller.scanResults.length, 2);
    });
  });

  group('connectDevice', () {
    test('connect 호출 후 connectionState가 connected로 바뀐다', () async {
      final device = BluetoothDevice.fromId('AA:BB:CC:DD:EE:FF');
      controller.connectDevice(device);

      await Future.delayed(Duration.zero);

      expect(
        controller.connectionState.value,
        BluetoothConnectionState.connected,
      );
    });
  });
}

List<ScanResult> _makeFakeScanResults(List<String> ids) {
  return ids
      .map((id) => ScanResult(
            device: BluetoothDevice.fromId(id),
            advertisementData: AdvertisementData.empty,
            rssi: -60,
            timeStamp: DateTime.now(),
          ))
      .toList();
}
```

![BLE 스캔 테스트 구조](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## 삽질했던 부분

`ScanResult`를 직접 생성하려고 하면 생성자가 없다. `flutter_blue_plus` 내부 구현이 플러그인 채널에서 역직렬화하는 구조라, 외부에서 임의로 만들 수가 없다.

해결 방법은 두 가지다.

**방법 1**: `flutter_blue_plus`의 공식 Mocking 채널을 쓰는 것. 패키지 내부 MethodChannel에 fake 응답을 주입해서 `ScanResult`를 직렬화된 형태로 만든다. 정확하지만 iot_deviceplate가 많다.

**방법 2**: `scanResults`를 Controller에서 `List<ScanResult>`로 가공하지 않고, Wrapper에서 이미 내부 데이터 모델(`BleDevice`)로 변환해서 넘긴다. Controller는 `BleDevice`만 다루면 되니까 `ScanResult` 생성 문제 자체가 없어진다.

실제 프로젝트에서는 방법 2를 택했다. Wrapper 레이어에서 `ScanResult → BleDevice`로 변환하면 Controller가 `flutter_blue_plus` 타입에 전혀 의존하지 않는다. 나중에 패키지를 바꿔도 Controller 코드는 건드릴 필요가 없다.

```dart
// 내부 도메인 모델
class BleDevice {
  final String id;
  final String name;
  final int rssi;
  const BleDevice({required this.id, required this.name, required this.rssi});
}

// Wrapper에서 변환
abstract class BleService {
  Stream<List<BleDevice>> get scanResults; // ScanResult 대신 BleDevice
  // ...
}
```

이렇게 하면 Fake에서 `BleDevice`를 직접 생성할 수 있다.

---

다음 편은 MQTT `mqtt5_client` 연결 로직의 Unit Test다. BLE와 다르게 연결 상태가 Stream이 아니라 콜백 기반이라서 또 다른 격리 방법이 필요하다.

**참고 링크**
- [flutter_blue_plus pub.dev](https://pub.dev/packages/flutter_blue_plus)
- [flutter_blue_plus Mocking Guide](https://pub.dev/packages/flutter_blue_plus#mocking)
- [GitHub Issue: How to mock FlutterBluePlus](https://github.com/boskokg/flutter_blue_plus/issues/556)
