---
layout: post
title: "Flutter Platform Channel 테스트 — MethodChannel Mock으로 플러그인 격리하기"
description: "flutter_test의 TestDefaultBinaryMessenger.setMockMethodCallHandler로 BLE 권한, 배터리, 네이티브 API를 실기기 없이 unit/widget 테스트에서 격리하는 방법을 실제 코드로 정리했다."
date: 2026-06-27
tags: [Flutter, Dart, flutter_blue_plus, Android, iOS]
comments: true
share: true
---

![Flutter 플랫폼 채널과 네이티브 레이어 테스트](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

IoT 앱을 테스트하다 보면 어느 순간 반드시 벽에 부딪힌다. BLE 권한 요청, 배터리 상태 조회, 네이티브 알림 스케줄링… 이런 건 전부 Platform Channel을 통해 네이티브 레이어와 통신한다. 그리고 테스트 환경에서는 그 네이티브 레이어가 없다.

처음엔 `MethodChannel.setMockMethodCallHandler`를 쓰면 되는 줄 알았다. 근데 Flutter 최신 버전에서 deprecated 경고가 뜨고, 어떤 걸로 바꿔야 하는지 공식 문서가 생각보다 불친절하다. 이 글에서는 `TestDefaultBinaryMessenger.setMockMethodCallHandler`로의 이전 방법과 실제 IoT 앱에서 쓰는 패턴을 정리한다.

## 왜 갑자기 deprecated가 됐나

예전에는 이렇게 썼다.

```dart
// 옛날 방식 — deprecated
MethodChannel('plugins.flutter.io/permission_handler').setMockMethodCallHandler(
  (call) async {
    if (call.method == 'checkPermissionStatus') return 1; // granted
    return null;
  },
);
```

Flutter 팀이 이 API를 `package:flutter`에서 `package:flutter_test`로 옮겼다. 이유는 명확하다. 테스트용 API가 프로덕션 패키지에 있으면 실수로 프로덕션 코드에서 호출할 수 있기 때문이다. 이전 API는 아직 동작하긴 하지만 deprecation 경고가 나오고, 향후 제거될 예정이다.

새 위치는 `TestDefaultBinaryMessenger`다.

## 새 API로 이전하는 방법

```dart
// widget test 안에서
testWidgets('권한 허용 시 BLE 스캔 시작', (tester) async {
  // tester.binding.defaultBinaryMessenger로 접근
  tester.binding.defaultBinaryMessenger.setMockMethodCallHandler(
    const MethodChannel('flutter.baseflow.com/permissions/methods'),
    (call) async {
      if (call.method == 'checkPermissionStatus') return 1; // PermissionStatus.granted
      if (call.method == 'requestPermissions') return {1: 1};
      return null;
    },
  );

  await tester.pumpWidget(const MyApp());
  await tester.tap(find.byType(BleScanButton));
  await tester.pump();

  expect(find.text('스캔 중...'), findsOneWidget);
});
```

Widget tester가 없는 순수 unit test에서는 바인딩에 직접 접근한다.

```dart
// unit test 안에서
setUp(() {
  TestWidgetsFlutterBinding.ensureInitialized();
  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockMethodCallHandler(
    const MethodChannel('flutter.baseflow.com/permissions/methods'),
    (call) async => 1, // 항상 granted
  );
});
```

`TestWidgetsFlutterBinding.ensureInitialized()` 없이 바인딩에 접근하면 null 오류가 나온다. 이것 때문에 반나절 날렸다.

## IoT 앱 실전 패턴 — BLE 권한 Mock

내가 쓰는 IoT 앱에서 `permission_handler`는 이렇게 생겼다.

```dart
// lib/infrastructure/permission/ble_permission_service.dart
class BlePermissionService {
  Future<bool> requestBlePermission() async {
    final status = await Permission.bluetoothScan.status;
    if (status.isGranted) return true;
    
    final result = await Permission.bluetoothScan.request();
    return result.isGranted;
  }
}
```

이 서비스를 테스트할 때 `Permission.bluetoothScan`은 내부적으로 `MethodChannel`을 호출한다. 실기기가 없으면 `MissingPluginException`이 뜬다.

```dart
// test/infrastructure/permission/ble_permission_service_test.dart
void main() {
  late BlePermissionService sut;
  const _channel = MethodChannel('flutter.baseflow.com/permissions/methods');

  setUp(() {
    TestWidgetsFlutterBinding.ensureInitialized();
    sut = BlePermissionService();
  });

  tearDown(() {
    // 핸들러 반드시 정리. 안 하면 다른 테스트에 오염된다.
    TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
        .setMockMethodCallHandler(_channel, null);
  });

  test('권한이 이미 허용된 경우 true 반환', () async {
    TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
        .setMockMethodCallHandler(_channel, (call) async {
      if (call.method == 'checkPermissionStatus') return 1; // granted
      return null;
    });

    final result = await sut.requestBlePermission();
    expect(result, isTrue);
  });

  test('권한 요청 후 거부 시 false 반환', () async {
    TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
        .setMockMethodCallHandler(_channel, (call) async {
      if (call.method == 'checkPermissionStatus') return 0; // denied
      if (call.method == 'requestPermissions') return {1: 0}; // denied
      return null;
    });

    final result = await sut.requestBlePermission();
    expect(result, isFalse);
  });
}
```

`tearDown`에서 핸들러를 null로 초기화하지 않으면 다른 테스트 케이스에 mock이 누출된다. 이건 반드시 해줘야 한다.

## EventChannel도 Mock할 수 있다

BLE 스캔 결과처럼 Stream으로 데이터를 받는 경우는 `EventChannel`을 쓴다. 이것도 Mock 가능하다.

```dart
// EventChannel mock — Stream 데이터 주입
void _mockEventChannel(String channelName, List<dynamic> events) {
  const codec = StandardMethodCodec();

  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockStreamHandler(
    EventChannel(channelName),
    MockStreamHandler.inline(
      onListen: (args, sink) {
        for (final event in events) {
          sink.success(event);
        }
        sink.endOfStream();
      },
    ),
  );
}

// 사용 예
test('BLE 스캔 결과가 Stream으로 들어온다', () async {
  _mockEventChannel('flutter_blue_plus/scan_results', [
    {'deviceId': 'AA:BB:CC:DD:EE:FF', 'rssi': -65},
    {'deviceId': '11:22:33:44:55:66', 'rssi': -80},
  ]);

  final results = await bleScanner.scanStream.take(2).toList();
  expect(results.length, 2);
  expect(results.first.deviceId, 'AA:BB:CC:DD:EE:FF');
});
```

`MockStreamHandler.inline`은 `flutter_test` 1.0.0 이상에서 사용 가능하다.

![MethodChannel과 EventChannel의 테스트 격리 구조](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## 삽질 포인트 — fakeAsync 안에서 async handler

`fakeAsync` 블록 안에서 async mock handler를 쓰면 테스트가 멈춘다.

```dart
// 이렇게 하면 deadlock
fakeAsync((fake) {
  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockMethodCallHandler(_channel, (call) async {
    // async handler + fakeAsync = 위험
    await Future.delayed(Duration.zero);
    return 1;
  });

  sut.requestBlePermission();
  fake.flushMicrotasks(); // 여기서 멈춤
});
```

공식 문서에도 명시되어 있다. `fakeAsync` 안에서는 mock handler를 **동기(synchronous)**로 작성해야 한다.

```dart
// 올바른 방법
fakeAsync((fake) {
  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockMethodCallHandler(_channel, (call) async {
    // async 키워드는 유지하되, 내부는 동기로
    return 1; // Future.delayed 없음
  });

  sut.requestBlePermission();
  fake.flushMicrotasks(); // 정상 동작
});
```

## 플러그인별 채널 이름 찾는 법

Mock을 설정하려면 플러그인이 쓰는 채널 이름을 정확히 알아야 한다.

```bash
# 플러그인 소스에서 채널 이름 찾기
find ~/.pub-cache/hosted/pub.dev/permission_handler-*/android/src -name "*.kt" | \
  xargs grep "MethodChannel\|CHANNEL_NAME" 2>/dev/null | head -5

# 또는 Dart 레이어에서
find ~/.pub-cache/hosted/pub.dev/permission_handler-*/lib -name "*.dart" | \
  xargs grep "MethodChannel\|const _channel" 2>/dev/null | head -5
```

채널 이름은 플러그인 버전마다 바뀔 수 있다. 내가 쓰는 패턴은 테스트 헬퍼 파일에 채널 이름을 상수로 모아두는 것이다.

```dart
// test/helpers/channel_names.dart
abstract class ChannelNames {
  static const permissionHandler = 'flutter.baseflow.com/permissions/methods';
  static const batteryInfo = 'samples.flutter.dev/battery';
  // 필요할 때마다 여기에 추가
}
```

이렇게 하면 플러그인 버전이 바뀌어서 채널명이 변경될 때 한 곳만 수정하면 된다.

---

Platform Channel mock은 IoT 앱 테스트에서 처음엔 번거롭게 느껴지지만, 한 번 패턴을 잡아두면 실기기 없이도 권한 분기, 에러 케이스, 다양한 네이티브 응답 시나리오를 빠르게 검증할 수 있다. 특히 CI 환경에서는 실기기가 없으니 이 패턴 없이는 네이티브 레이어 관련 코드를 전혀 테스트하지 못한다.

다음 편은 GetX Controller와 UseCase 레이어를 함께 묶어서 테스트하는 **통합 단위 테스트 설계** 패턴을 다룬다.

**참고 링크**
- [Flutter 공식 — Platform Channel 테스트 인터페이스 이전](https://docs.flutter.dev/release/breaking-changes/mock-platform-channels)
- [TestDefaultBinaryMessenger API 레퍼런스](https://api.flutter.dev/flutter/flutter_test/TestDefaultBinaryMessenger-class.html)
- [flutter/testing/testing-plugins 공식 가이드](https://docs.flutter.dev/testing/testing-plugins)
