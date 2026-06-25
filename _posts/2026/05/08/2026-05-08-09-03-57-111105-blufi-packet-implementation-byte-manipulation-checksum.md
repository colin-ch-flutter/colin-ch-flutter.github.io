---
layout: post
title: "BluFi 패킷 직접 구현하기 - 바이트 조작과 체크섬"
description: " "
date: 2026-05-08
tags: [Flutter, BLE, BluFi, 패킷, Dart]
comments: true
share: true
---
![바이너리 코드](https://images.unsplash.com/photo-1526374965328-7f61d4dc18c5?w=800&q=80)

## Dart로 바이트 다루기

Dart에서 바이트 배열을 다룰 때는 `Uint8List`를 쓴다. `List<int>`와 비슷하지만 0~255 범위의 정수만 담는 고정 크기 배열이다:

```dart
import 'dart:typed_data';

// 바이트 배열 생성
final buffer = Uint8List(10);

// 특정 위치에 값 쓰기
buffer[0] = 0x01;
buffer[1] = 0xFF;

// ByteData로 다중 바이트 값 처리
final byteData = ByteData.sublistView(buffer);
byteData.setUint16(2, 0x1234, Endian.big);  // 빅 엔디언으로 쓰기

// 16진수로 출력 (디버깅용)
print(buffer.map((b) => '0x${b.toRadixString(16).padLeft(2, '0')}').join(' '));
```

## BluFi 패킷 생성

```dart
// blufi_codec.dart
class BlufiCodec {
  int _sequence = 0;

  Uint8List buildPacket({
    required int type,
    required int subtype,
    Uint8List? data,
    bool frag = false,
    int totalLen = 0,
  }) {
    final dataLen = data?.length ?? 0;
    // frag이면 total length 2바이트 추가
    final fragHeaderLen = frag ? 2 : 0;
    final packetSize = 4 + fragHeaderLen + dataLen + 2; // header + data + checksum

    final packet = Uint8List(packetSize);
    var offset = 0;

    // Type 바이트: 상위 2비트 = type, 하위 6비트 = subtype
    packet[offset++] = (subtype << 2) | type;

    // Frame Control
    int fc = 0;
    if (frag) fc |= 0x04;  // 분할 패킷 플래그
    packet[offset++] = fc;

    // Sequence (0~255 순환)
    packet[offset++] = _sequence++ & 0xFF;

    // Data Length (분할 헤더 포함)
    packet[offset++] = dataLen + fragHeaderLen;

    // 분할 패킷이면 총 길이 먼저
    if (frag) {
      packet[offset++] = totalLen & 0xFF;
      packet[offset++] = (totalLen >> 8) & 0xFF;
    }

    // 실제 데이터
    if (data != null) {
      packet.setRange(offset, offset + dataLen, data);
      offset += dataLen;
    }

    // 체크섬 계산 (CRC-16)
    final checksum = _calculateCrc(packet, 0, offset);
    packet[offset++] = checksum & 0xFF;
    packet[offset++] = (checksum >> 8) & 0xFF;

    return packet;
  }

  int _calculateCrc(Uint8List data, int start, int end) {
    int crc = 0;
    for (int i = start; i < end; i++) {
      crc ^= data[i];
      for (int j = 0; j < 8; j++) {
        if (crc & 1 != 0) {
          crc = (crc >> 1) ^ 0xA001;
        } else {
          crc >>= 1;
        }
      }
    }
    return crc;
  }
}
```

## WiFi SSID 패킷 만들기

실제로 SSID를 보내는 패킷을 만드는 예시:

```dart
Uint8List buildSsidPacket(String ssid) {
  final ssidBytes = Uint8List.fromList(utf8.encode(ssid));
  return buildPacket(
    type: 1,        // Data frame
    subtype: 0x02,  // SSID subtype
    data: ssidBytes,
  );
}
```

## 분할 전송 처리

MTU 크기 제한 때문에 데이터가 크면 여러 패킷으로 쪼개서 보내야 한다:

```dart
Future<void> sendWithFragmentation(
  BluetoothCharacteristic char,
  int type,
  int subtype,
  Uint8List data,
  int mtu,
) async {
  // 헤더 크기를 제외한 실제 데이터 최대 크기
  final maxDataPerPacket = mtu - 6;  // 헤더 4 + 체크섬 2

  if (data.length <= maxDataPerPacket) {
    // 단일 패킷으로 전송
    final packet = buildPacket(type: type, subtype: subtype, data: data);
    await char.write(packet, withoutResponse: false);
    return;
  }

  // 분할 전송
  int offset = 0;
  final totalLen = data.length;

  while (offset < data.length) {
    final chunkSize = min(maxDataPerPacket - 2, data.length - offset); // -2: frag header
    final chunk = data.sublist(offset, offset + chunkSize);
    final isFirst = offset == 0;

    final packet = buildPacket(
      type: type,
      subtype: subtype,
      data: chunk,
      frag: true,
      totalLen: isFirst ? totalLen : 0,
    );

    await char.write(packet, withoutResponse: false);
    offset += chunkSize;

    // 패킷 간 약간의 딜레이 (기기가 처리할 시간)
    await Future.delayed(const Duration(milliseconds: 10));
  }
}
```

## 수신 패킷 조립기

분할된 패킷을 다시 조립하는 게 `blufi_packet_assembler.dart`의 역할이다:

```dart
class BlufiPacketAssembler {
  final List<Uint8List> _fragments = [];
  int _expectedTotal = 0;
  int _receivedBytes = 0;

  // 패킷을 받아서 완성된 데이터가 있으면 반환
  Uint8List? add(Uint8List packet) {
    final isFragment = (packet[1] & 0x04) != 0;

    if (!isFragment) {
      // 단일 패킷이면 바로 반환
      return extractData(packet);
    }

    // 첫 번째 분할 패킷에서 총 길이 읽기
    if (_fragments.isEmpty) {
      _expectedTotal = packet[4] | (packet[5] << 8);
    }

    final data = extractData(packet);
    _fragments.add(data);
    _receivedBytes += data.length;

    if (_receivedBytes >= _expectedTotal) {
      // 모든 조각이 도착했으면 합치기
      final result = Uint8List(_receivedBytes);
      int pos = 0;
      for (final frag in _fragments) {
        result.setRange(pos, pos + frag.length, frag);
        pos += frag.length;
      }
      _reset();
      return result;
    }

    return null; // 아직 더 기다려야 함
  }

  void _reset() {
    _fragments.clear();
    _expectedTotal = 0;
    _receivedBytes = 0;
  }
}
```

## 디버깅 팁

바이트 레벨 디버깅은 16진수로 출력하는 게 핵심이다:

```dart
void logBytes(String label, Uint8List data) {
  final hex = data.map((b) => b.toRadixString(16).padLeft(2, '0').toUpperCase()).join(' ');
  print('[$label] ${data.length} bytes: $hex');
}
```

---

다음 편은 이 모든 걸 종합해서 WiFi 프로비저닝 전체 플로우를 완성하는 과정이다.
