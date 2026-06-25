---
layout: post
title: "BLE WiFi 프로비저닝 완성 - 기기 연결 프로세스 전체 흐름"
description: " "
date: 2026-05-09
tags: [Flutter, BLE, WiFi프로비저닝, BluFi, IoT]
comments: true
share: true
---
![네트워크 연결](https://images.unsplash.com/photo-1544197150-b99a580bb7a8?w=800&q=80)

## 전체 기기 등록 UX 흐름

사용자가 새 IoT 기기를 앱에 등록하는 과정을 UI 흐름으로 그려보면:

```
1. 기기 추가 버튼 클릭
2. 기기 준비 안내 화면 (IoT 기기 전원 확인 등)
3. BLE 스캔 화면 (주변 기기 목록 표시)
4. 기기 선택
5. WiFi SSID/비밀번호 입력
6. 연결 시도 중 (프로그레스 화면)
7. 닉네임 설정
8. 타임존 설정
9. 완료
```

각 단계가 별도 화면으로 분리되어 있고, 라우팅은 `app_pages.dart`에 모두 정의되어 있다.

## BluFi 연결 클라이언트 구조

`blufi_ble_client.dart`는 전체 프로비저닝 프로세스를 관리한다:

```dart
class BlufiBleClient {
  BluetoothDevice? _device;
  BluetoothCharacteristic? _writeChar;
  BluetoothCharacteristic? _notifyChar;
  final BlufiCodec _codec = BlufiCodec();
  final BlufiPacketAssembler _assembler = BlufiPacketAssembler();
  
  final StreamController<BlufiResult> _resultController =
      StreamController.broadcast();
  
  Stream<BlufiResult> get results => _resultController.stream;

  Future<void> connect(BluetoothDevice device) async {
    _device = device;
    await device.connect(timeout: const Duration(seconds: 10));
    await _discoverServices();
    await _setupNotify();
  }

  Future<void> provision({
    required String ssid,
    required String password,
  }) async {
    // 1. 네고시에이션
    await _negotiate();
    
    // 2. 운전 모드 설정 (Station 모드)
    await _sendOpMode(1);  // 1 = STA mode
    
    // 3. WiFi 정보 전송
    await _sendSsid(ssid);
    await _sendPassword(password);
    
    // 4. 연결 명령
    await _sendConnectCommand();
  }

  Future<void> _sendSsid(String ssid) async {
    final data = Uint8List.fromList(utf8.encode(ssid));
    final packet = _codec.buildPacket(
      type: 1,
      subtype: 0x02,
      data: data,
    );
    await _writeChar!.write(packet, withoutResponse: false);
  }
}
```

## 결과 대기와 타임아웃

WiFi 연결 결과를 기다리는 부분이 중요하다. IoT 기기가 WiFi에 연결되는 데 시간이 걸리는데, 이 동안 앱은 기다려야 한다:

```dart
Future<BlufiResult> waitForResult({
  Duration timeout = const Duration(seconds: 30),
}) async {
  try {
    return await results
        .where((r) => r.type == BlufiResultType.wifiConnect)
        .first
        .timeout(timeout);
  } on TimeoutException {
    throw BlufiException('WiFi 연결 대기 시간 초과');
  }
}
```

## 연결 실패 처리

WiFi 비밀번호가 틀렸거나, 신호가 약하거나, 다양한 이유로 연결이 실패할 수 있다. 실패 원인에 따라 다른 안내를 보여준다:

```dart
// 기기에서 오는 연결 결과 코드
enum WifiConnectResult {
  success(0),
  passwordWrong(2),
  apNotFound(3),
  timeout(4),
  unknown(255);
  
  const WifiConnectResult(this.code);
  final int code;
  
  static WifiConnectResult fromCode(int code) =>
      values.firstWhere((e) => e.code == code, orElse: () => unknown);
}

String getErrorMessage(WifiConnectResult result) {
  return switch (result) {
    WifiConnectResult.passwordWrong => 'WiFi 비밀번호가 맞지 않습니다',
    WifiConnectResult.apNotFound => 'WiFi 신호를 찾을 수 없습니다. IoT 기기 주변에서 시도해보세요',
    WifiConnectResult.timeout => '연결 시간이 초과되었습니다',
    _ => '연결에 실패했습니다. 다시 시도해주세요',
  };
}
```

## 연결 미완료 상태 처리

WiFi 연결까지는 성공했지만 서버 등록이 실패한 경우, 앱을 재실행했을 때 기기가 "연결 미완료" 상태로 표시된다. `DeviceConnectIncompleteView`가 이 상황을 처리한다.

이런 엣지 케이스를 생각해두지 않으면 사용자가 "분명히 연결 성공했는데 기기가 안 보인다"는 문의를 해온다.

## 완성된 흐름의 UX 포인트

기기 등록은 사용자가 처음 겪는 경험이라 특히 중요하다. 실제로 적용한 UX 원칙들:

1. **각 단계에서 진행 상황을 명확히 표시**: "WiFi 연결 중... (보통 20-30초 소요)" 같은 문구
2. **실패했을 때 이유를 알려준다**: 막연한 "실패" 대신 원인별 안내
3. **재시도를 쉽게**: 실패 화면에서 처음으로 돌아가는 버튼을 명확하게
4. **성공했을 때 명확한 피드백**: 완료 화면에 애니메이션 추가

---

다음 편부터는 MQTT 파트가 시작된다. BLE로 기기를 WiFi에 연결했으니, 이제 클라우드를 통해 원격 제어하는 부분이다.
