---
layout: post
title: "Flutter Widget Previewer - IoT 제어 카드 상태를 빠르게 확인하는 방법"
description: "Flutter 3.47에서 stable이 된 Widget Previewer로 IoT 제어 카드의 켜짐·꺼짐·오프라인 상태를 앱 전체 실행 없이 확인하고, preview 전용 의존성 분리 기준까지 정리했다."
date: 2026-08-31
tags: [Flutter, IoT, 스마트홈, UI/UX, 테스트]
comments: true
share: true
---

![Flutter Widget Previewer로 IoT 스마트홈 제어 카드 확인](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

이 그림에서 봐야 할 부분은 집 전체 화면이 아니라 조명·보일러 카드 하나의 상태를 빠르게 바꾸며 확인하는 흐름이다.

Flutter Widget Previewer는 앱을 전체 실행하지 않고 개별 Widget을 실시간으로 확인하는 도구다. Flutter 3.47부터 stable이 됐고, `flutter widget-preview start`로 로컬 미리보기 환경을 열 수 있다. IoT 앱에서는 MQTT 브로커를 연결하고 로그인까지 거쳐야 카드가 보이는 경우가 많았는데, 이 도구를 붙이니 UI 확인에 필요한 대기 시간이 거의 사라졌다.

## 앱 전체 실행이 오히려 느렸던 이유

처음엔 스마트홈 대시보드를 실행한 뒤 실제 MQTT 상태를 받아 카드 모양을 확인했다. 해보니 UI를 한 줄 고칠 때마다 인증 복원, Realm 초기화, MQTT 연결을 모두 기다려야 했다. 더 나쁜 점은 브로커가 오프라인이면 카드의 `offline` 상태를 확인하기도 전에 화면이 멈춘다는 것이다.

상태를 카드의 입력값으로 만들고, Preview에서는 Fake Repository만 주입했다.

| 확인할 상태 | 앱 전체 실행 | Widget Preview |
|---|---|---|
| 보일러 켜짐 | 인증·MQTT 필요 | 즉시 확인 |
| 기기 오프라인 | 네트워크 상황에 의존 | 고정된 Fake 상태 |
| 긴 기기명·오류 문구 | 특정 데이터 준비 필요 | preview 이름으로 분리 |
| 다크모드 | 앱 설정 변경 필요 | 밝기별 preview 제공 |

## 상태별 Preview를 따로 만든다

Preview 함수는 필수 인자가 없는 Widget이나 Widget을 반환하는 top-level 함수에 붙일 수 있다. 아래처럼 카드 자체는 실제 화면과 공유하고, 데이터만 고정한다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter/widget_previews.dart';

enum DeviceStatus { on, off, offline }

class DeviceCard extends StatelessWidget {
  const DeviceCard({required this.name, required this.status, super.key});

  final String name;
  final DeviceStatus status;

  @override
  Widget build(BuildContext context) {
    final label = switch (status) {
      DeviceStatus.on => '켜짐',
      DeviceStatus.off => '꺼짐',
      DeviceStatus.offline => '오프라인',
    };
    return Card(
      child: ListTile(
        title: Text(name),
        subtitle: Text(label),
        trailing: Icon(
          status == DeviceStatus.on ? Icons.power : Icons.power_off,
        ),
      ),
    );
  }
}

@Preview(name: '거실 보일러 - 켜짐', group: 'DeviceCard')
Widget boilerOnPreview() => const DeviceCard(
      name: '거실 보일러',
      status: DeviceStatus.on,
    );

@Preview(name: '기기 오프라인', group: 'DeviceCard')
Widget deviceOfflinePreview() => const DeviceCard(
      name: '현관 조명',
      status: DeviceStatus.offline,
    );
```

실행 명령은 프로젝트 루트에서 다음 한 줄이다.

```bash
flutter widget-preview start
```

Preview 화면에서는 이름이나 그룹으로 상태를 필터링할 수 있고, 밝은 테마와 어두운 테마를 바로 전환할 수 있다. 긴 이름, 큰 글자, 작은 화면까지 같이 보고 싶다면 `size`, `textScaleFactor`, `brightness`를 Preview에 지정하면 된다.

## Fake를 어디까지 넣을지

Preview 함수 안에서 `MqttClient`나 `dart:io`를 직접 만들면 안 된다. Widget Previewer는 Flutter Web 환경에서 동작하므로 네이티브 플러그인과 `dart:io`, `dart:ffi` API를 사용할 수 없다. 카드가 서비스에 직접 접근하는 구조라면 Preview를 붙이는 순간 의존성 문제가 드러난다.

나는 `DeviceCard`를 순수 표시 Widget으로 두고, MQTT·BLE 상태 변환은 Controller와 Repository에서 끝내도록 분리했다. Preview에서 필요한 것은 `DeviceViewData`뿐이다. 이 경계를 만들고 나니 Widget Test에도 같은 fixture를 재사용할 수 있었다.

주의할 점은 Preview가 실제 BLE 연결, MQTT 재연결, 권한 팝업을 검증해주지 않는다는 것이다. 그것들은 integration_test의 책임으로 남겨야 한다. Preview는 상태 조합과 레이아웃을 빠르게 확인하고, 통신과 플랫폼 동작은 실제 테스트에서 확인하는 식으로 역할을 나누는 편이 맞다.

짧게 정리하면 Flutter Widget Previewer는 IoT 앱의 통신 테스트를 대신하는 도구가 아니다. 대신 `켜짐·꺼짐·오프라인·긴 텍스트·다크모드`처럼 화면 상태가 많은 카드의 반복 확인 비용을 크게 줄여준다. Flutter 3.47 이상이라면 작은 Widget부터 Preview를 붙이고, 네이티브 의존성은 Preview 바깥의 Repository 경계에 남겨두는 것이 시작점이다.

공식 문서: [Flutter Widget Previewer](https://docs.flutter.dev/tools/widget-previewer)
