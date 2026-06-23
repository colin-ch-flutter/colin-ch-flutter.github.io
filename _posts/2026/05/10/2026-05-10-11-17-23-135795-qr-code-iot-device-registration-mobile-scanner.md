---
layout: post
title: "QR 코드로 IoT 기기 등록하기 - mobile_scanner 완벽 활용"
description: " "
date: 2026-05-10
tags: [Flutter, QR코드, mobile_scanner, IoT, 기기등록]
comments: true
share: true
---

# QR 코드로 IoT 기기 등록하기 - mobile_scanner 완벽 활용

![QR 스캔](https://images.unsplash.com/photo-1567446537708-ac4aa75c9c28?w=800&q=80)

## QR 방식이 BLE보다 나은 경우

BLE 방식의 기기 등록은 처음 소개할 때 설명했다. 그런데 BLE만으로는 해결이 안 되는 상황이 있다.

예를 들어 IoT 모니터링 대시보드 방식의 상업용 IoT 기기는 BluFi를 지원하지 않을 수 있다. 또는 기기가 이미 WiFi에 연결되어 있고, 기기 ID만 앱에 등록하면 되는 경우도 있다.

이때 QR 코드 등록이 유용하다. IoT 기기 본체나 설명서에 QR 코드가 프린트되어 있고, 앱으로 스캔하면 기기 ID를 읽어서 바로 등록하는 방식이다.

## mobile_scanner 설정

`pubspec.yaml`:
```yaml
dependencies:
  mobile_scanner: ^7.1.4
```

### Android 권한

```xml
<uses-permission android:name="android.permission.CAMERA" />
```

### iOS 권한

```xml
<key>NSCameraUsageDescription</key>
<string>기기 등록을 위해 QR 코드를 스캔합니다</string>
```

## 스캔 화면 구현

```dart
class DeviceConnectQrView extends StatelessWidget {
  const DeviceConnectQrView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Stack(
        children: [
          MobileScanner(
            onDetect: (capture) {
              final barcodes = capture.barcodes;
              if (barcodes.isEmpty) return;
              
              final qrValue = barcodes.first.rawValue;
              if (qrValue != null) {
                _handleQrResult(qrValue);
              }
            },
          ),
          // 스캔 영역 오버레이
          const QrScanOverlay(),
          // 뒤로가기 버튼
          Positioned(
            top: 48,
            left: 16,
            child: IconButton(
              icon: const Icon(Icons.close, color: Colors.white),
              onPressed: () => Get.back(),
            ),
          ),
        ],
      ),
    );
  }

  void _handleQrResult(String value) {
    // QR 값에서 기기 ID 파싱
    final deviceId = parseDeviceId(value);
    if (deviceId == null) {
      // 잘못된 QR 코드
      Get.snackbar('오류', '유효하지 않은 QR 코드입니다');
      return;
    }
    
    // 기기 등록 화면으로 이동
    Get.toNamed(Routes.deviceConnectNickname, arguments: {'deviceId': deviceId});
  }
}
```

## QR 스캔 영역 오버레이

카메라 전체 화면보다 스캔 영역을 표시해주는 게 UX에 좋다:

```dart
class QrScanOverlay extends StatelessWidget {
  const QrScanOverlay({super.key});

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      painter: _ScanOverlayPainter(),
      child: const SizedBox.expand(),
    );
  }
}

class _ScanOverlayPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    final scanArea = Rect.fromCenter(
      center: Offset(size.width / 2, size.height / 2 - 50),
      width: 250,
      height: 250,
    );

    // 주변 어둡게
    final paint = Paint()..color = Colors.black.withOpacity(0.5);
    canvas.drawPath(
      Path.combine(
        PathOperation.difference,
        Path()..addRect(Offset.zero & size),
        Path()..addRRect(RRect.fromRectAndRadius(scanArea, const Radius.circular(12))),
      ),
      paint,
    );

    // 스캔 영역 테두리
    canvas.drawRRect(
      RRect.fromRectAndRadius(scanArea, const Radius.circular(12)),
      Paint()
        ..color = Colors.white
        ..style = PaintingStyle.stroke
        ..strokeWidth = 2,
    );
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) => false;
}
```

## QR 값 파싱

QR 코드에는 기기 ID만 있을 수도 있고, 여러 정보가 JSON 형태로 들어있을 수도 있다:

```dart
String? parseDeviceId(String qrValue) {
  // 단순 ID 형식
  if (RegExp(r'^[A-Z0-9]{16}$').hasMatch(qrValue)) {
    return qrValue;
  }
  
  // JSON 형식
  try {
    final json = jsonDecode(qrValue);
    return json['deviceId'] as String?;
  } catch (_) {
    return null;
  }
}
```

## 중복 스캔 방지

카메라가 연속으로 같은 QR을 읽으면 핸들러가 여러 번 호출될 수 있다. 한 번만 처리되도록 플래그를 추가한다:

```dart
bool _isProcessing = false;

void _handleQrResult(String value) {
  if (_isProcessing) return;
  _isProcessing = true;
  
  // 처리 로직...
  
  // 3초 후 다시 스캔 가능
  Future.delayed(const Duration(seconds: 3), () {
    _isProcessing = false;
  });
}
```

---

다음 편은 BLE 연결 안정성을 높이는 재연결 전략과 오류 처리다.
