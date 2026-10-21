---
layout: post
title: "Flutter iOS 26 BLE 오류 - 백그라운드 state restoration이 다시 실행되지 않을 때"
description: "Flutter BLE 앱이 iOS 26에서 백그라운드 복귀나 연결 복원을 놓칠 때 bluetooth-central만 추가하면 안 되는 이유와 AccessorySetupKit, CoreBluetooth 상태 복구 조건을 정리한다."
date: 2026-10-21
tags: [Flutter, iOS, BLE, AccessorySetupKit, CoreBluetooth, 출시]
comments: true
share: true
---

![iOS 26 Flutter BLE 앱과 연결된 스마트 액세서리](https://images.unsplash.com/photo-1606220945770-b5b6c2c55bf1?auto=format&fit=crop&w=1600&q=80)

그림에서 볼 부분은 화면이 아니라, 앱이 닫힌 뒤에도 액세서리 연결 이벤트를 다시 받을 수 있는 운영 조건이다.

Flutter BLE 앱이 iOS 26에서 화면을 잠근 뒤 기기 알림을 놓친다면 `UIBackgroundModes`에 `bluetooth-central`만 추가하는 방식으로는 부족하다. 일반적인 `CoreBluetooth` 상태 보존은 여전히 필요하지만, iOS 26부터 시스템이 백그라운드 재실행 대상으로 보는 앱은 `AccessorySetupKit`으로 액세서리를 설정한 앱으로 좁아졌다. 단순 스캔 앱이나 사용자가 앱을 강제 종료한 경우까지 항상 복귀한다고 약속하면 안 된다.

## 기존 설정이 iOS 26에서 부족한 이유

`bluetooth-central`은 BLE 이벤트를 백그라운드에서 처리할 수 있게 하는 선언이지, 앱을 언제나 되살리는 보증서가 아니다. Apple의 TN3115 기준으로 앱이 메모리에서 제거된 경우에는 복귀할 수 있지만, 사용자가 앱 전환기에서 강제 종료하면 복귀하지 않을 수 있다. iOS 26의 변경점은 이 복귀 후보를 `AccessorySetupKit`으로 설정한 앱 중심으로 제한했다는 데 있다.

| 앱 구성 | iOS 26에서 기대할 수 있는 범위 | 출시 판단 |
|---|---|---|
| 일반 BLE 스캔 + `bluetooth-central` | 백그라운드 이벤트 처리 가능, 재실행은 조건부 | 연결 복구를 제품 보장으로 쓰지 않는다 |
| CoreBluetooth 상태 보존 + 서비스 알림 구독 | 시스템이 보존한 연결·알림 이벤트 복구 | 복귀 시 초기화 순서를 검증한다 |
| AccessorySetupKit으로 액세서리 설정 | iOS 26의 재실행 대상 조건에 부합 | 액세서리 설정 UX와 네이티브 연동이 필요하다 |
| 사용자가 앱을 직접 강제 종료 | 시스템 재실행을 기대할 수 없음 | 앱 재실행 후 수동 재연결을 제공한다 |

## Flutter 프로젝트에서 나눠야 할 설정

Flutter의 Dart 코드만으로 AccessorySetupKit 세션을 만들 수는 없다. 사용하는 BLE 플러그인이 해당 API를 노출하지 않는다면 Swift 플랫폼 채널이나 AccessorySetupKit을 지원하는 플러그인 계층을 별도로 둬야 한다. 반면 기존 CoreBluetooth 연결을 복원하는 최소 선언은 `Info.plist`에서 확인할 수 있다.

아래 설정은 일반 BLE 백그라운드 모드와 액세서리 설정 선언을 구분해 보여준다. 서비스 UUID와 이름은 실제 제품 값으로 바꿔야 한다.

```xml
<!-- ios/Runner/Info.plist -->
<key>UIBackgroundModes</key>
<array>
    <string>bluetooth-central</string>
</array>

<!-- AccessorySetupKit을 사용하는 경우에만 추가한다. -->
<key>NSAccessorySetupKitSupports</key>
<array>
    <string>Bluetooth</string>
</array>
<key>NSAccessorySetupBluetoothServices</key>
<array>
    <string>12345678-1234-1234-1234-1234567890AB</string>
</array>
<key>NSAccessorySetupBluetoothNames</key>
<array>
    <string>MyDevice</string>
</array>
```

CoreBluetooth 쪽에서는 `CBCentralManagerOptionRestoreIdentifierKey`로 복원 식별자를 고정하고, 앱이 백그라운드에서 시작될 때도 중앙 관리자를 너무 늦게 만들지 않아야 한다. 이 초기화는 Dart 화면이 열린 뒤가 아니라 iOS 네이티브 생명주기에서 수행하는 편이 안전하다. 복원 이벤트를 받은 시점에 FlutterEngine이 준비되지 않았다면 이벤트를 임시 저장하고, Dart isolate가 올라온 후 한 번만 전달한다.

```swift
let options: [String: Any] = [
    CBCentralManagerOptionRestoreIdentifierKey: "com.example.ble.central",
    CBCentralManagerOptionShowPowerAlertKey: true
]

central = CBCentralManager(delegate: self, queue: nil, options: options)
```

Flutter 쪽에서는 “앱이 백그라운드에서 깨어났다”와 “기기가 연결됐다”를 같은 상태로 취급하지 않는 것이 핵심이다. 복원 콜백이 와도 블루투스가 꺼져 있거나 서비스 검색이 끝나지 않았을 수 있으므로 `restoring → discovering → connected`처럼 단계별 상태를 둔다. 복원 데이터를 곧바로 화면 위젯에 전달하면 초기 프레임에서 연결 해제 화면을 잠깐 보여주는 문제가 생길 수 있다.

## 출시 전에 재현할 체크리스트

- [ ] 실제 iPhone에서 `bluetooth-central`과 액세서리 설정 선언을 Release 설정으로 확인했다.
- [ ] 연결 후 특성 알림을 구독한 상태에서 화면 잠금과 앱 백그라운드를 각각 시험했다.
- [ ] 앱이 메모리에서 제거된 경우와 사용자가 앱 전환기에서 강제 종료한 경우를 분리했다.
- [ ] iOS 25와 iOS 26에서 동일한 복귀를 보장한다고 가정하지 않았다.
- [ ] 복원 콜백이 FlutterEngine보다 먼저 오면 이벤트를 버리지 않고 임시 보관한다.
- [ ] 복귀 실패 시 사용자가 앱 화면에서 수동 재연결할 수 있다.

액세서리를 한 번 연결하고 계속 감시해야 하는 앱이라면 AccessorySetupKit 도입 비용을 검토할 가치가 있다. 단순히 근처 기기를 한 번 선택하는 기능이라면 시스템 Photo Picker처럼 권한 없는 흐름으로 바꾸라는 뜻은 아니다. 이 글의 핵심은 `bluetooth-central`을 넣으면 앱이 반드시 살아난다는 오래된 가정을 iOS 26 출시 기준에서 제거하는 데 있다.

짧게 정리하면, Flutter BLE 앱의 iOS 26 복귀 문제는 Dart 재연결 로직만의 문제가 아니다. `bluetooth-central`, CoreBluetooth 복원 식별자, AccessorySetupKit 설정 여부, 사용자의 강제 종료를 각각 분리해 검증해야 한다. 실제 제품이 백그라운드 재실행을 약속해야 하는 경우에만 네이티브 설정과 플랫폼 채널 작업을 추가하고, 그렇지 않다면 복귀 실패를 전제로 한 수동 재연결 UX가 더 안전하다.

출처와 확인 기준: 2026-10-21. [Apple TN3115 Bluetooth State Restoration 재실행 규칙](https://developer.apple.com/documentation/technotes/tn3115-bluetooth-state-restoration-app-relaunch-rules), [Apple AccessorySetupKit 문서](https://developer.apple.com/documentation/accessorysetupkit), [Flutter background process 안내](https://docs.flutter.dev/packages-and-plugins/background-processes)
