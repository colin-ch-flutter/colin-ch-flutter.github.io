---
layout: post
title: "BLE 기기 발견 후 연결하기 - GATT 서비스와 특성값 이해"
description: " "
date: 2026-05-06
tags: [Flutter, BLE, GATT, flutter_blue_plus, IoT]
comments: true
share: true
---

# BLE 기기 발견 후 연결하기 - GATT 서비스와 특성값 이해

![블루투스 연결](https://images.unsplash.com/photo-1589254065878-42c9da997008?w=800&q=80)

## GATT란

BLE 기기와 데이터를 주고받으려면 GATT(Generic Attribute Profile)를 이해해야 한다. GATT는 BLE 통신의 데이터 구조 규약이다.

계층 구조는 이렇다:

```
Profile
└── Service (서비스)
    └── Characteristic (특성값)
        └── Descriptor (설명자)
```

**Service**: 기능의 묶음. 예를 들어 "Device Information Service"는 기기 정보를 담는 서비스다.

**Characteristic**: 실제 데이터. 각 Characteristic은 UUID로 식별한다. 읽기(Read), 쓰기(Write), 알림(Notify) 등의 속성을 가진다.

BluFi 프로토콜의 경우 특정 서비스 UUID와 Characteristic UUID를 통해 데이터를 주고받는다.

## 기기 연결

```dart
final device = scanResult.device;

// 연결
await device.connect(
  timeout: const Duration(seconds: 10),
  autoConnect: false,
);

// 연결 상태 구독
device.connectionState.listen((state) {
  if (state == BluetoothConnectionState.connected) {
    discoverServices();
  } else if (state == BluetoothConnectionState.disconnected) {
    handleDisconnect();
  }
});
```

`autoConnect: false`로 설정하면 연결 실패 시 자동으로 재시도하지 않는다. 수동으로 재시도 로직을 관리하는 게 더 예측 가능한 동작을 만든다.

## 서비스 탐색

연결 후 서비스를 탐색해서 원하는 Characteristic을 찾는다:

```dart
Future<void> discoverServices() async {
  final services = await device.discoverServices();
  
  for (BluetoothService service in services) {
    if (service.uuid == blufiServiceUuid) {
      for (BluetoothCharacteristic char in service.characteristics) {
        if (char.uuid == blufiWriteCharUuid) {
          writeChar = char;
        } else if (char.uuid == blufiNotifyCharUuid) {
          notifyChar = char;
        }
      }
    }
  }
}
```

## Notify 구독

IoT 기기에서 응답이 올 때 알림을 받으려면 Notify 구독을 해야 한다:

```dart
// Notify 활성화
await notifyChar.setNotifyValue(true);

// 데이터 수신
notifyChar.lastValueStream.listen((data) {
  // data는 List<int> 타입
  handleReceivedData(data);
});
```

## Write 구현

IoT 기기에 데이터를 보낼 때:

```dart
// WithoutResponse: 응답을 기다리지 않음 (빠름)
// WithResponse: 응답을 기다림 (신뢰성 높음)
await writeChar.write(
  data,
  withoutResponse: false,
);
```

BluFi 프로토콜에서는 `withoutResponse: false`로 각 패킷의 전송을 확인한다.

## MTU 크기 조정

BLE는 한 번에 보낼 수 있는 데이터 크기(MTU)가 제한되어 있다. 기본 MTU는 23바이트인데, 이것보다 큰 데이터를 보내려면 MTU를 늘리거나 데이터를 분할해야 한다:

```dart
// MTU 요청 (Android만 가능, iOS는 자동)
await device.requestMtu(512);
final mtu = await device.mtu.first;
print('협상된 MTU: $mtu bytes');
```

iOS는 MTU를 직접 요청할 수 없고 시스템이 자동으로 협상한다. 보통 185바이트 정도가 된다.

## 연결 후 상태 확인

연결이 됐다고 바로 데이터를 보내면 안 된다. `discoverServices()`가 완료된 후에 Characteristic에 접근해야 한다. 이걸 놓치면 `null` 참조 에러가 발생한다.

```dart
BluetoothCharacteristic? writeChar;
BluetoothCharacteristic? notifyChar;

bool get isReady => writeChar != null && notifyChar != null;
```

`isReady`를 체크한 후에 데이터를 보내도록 방어 코드를 추가하는 게 좋다.

---

다음 편은 BluFi 프로토콜에 대한 설명이다. 이 부분이 이 시리즈에서 가장 구현하기 까다로웠던 부분이다.
