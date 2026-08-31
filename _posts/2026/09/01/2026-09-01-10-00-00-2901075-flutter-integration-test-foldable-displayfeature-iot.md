---
layout: post
title: "Flutter integration_test 폴더블 화면 검증 - DisplayFeature와 힌지 레이아웃 테스트"
description: "Flutter integration_test에서 폴더블·듀얼 스크린의 DisplayFeature와 힌지 영역을 고려한 IoT 화면을 검증하고, 테스트에서 레이아웃 정보를 주입하는 방법을 정리했다."
date: 2026-09-01
tags: [Flutter, Dart, 테스트, integration_test, UI, IoT, Android]
comments: true
share: true
---

![Flutter integration_test 폴더블 스마트홈 대시보드 레이아웃](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

이 그림에서 볼 부분은 화면이 하나의 직사각형이라는 가정이 폴더블 기기에서는 깨진다는 점이다.

Flutter integration_test에서 폴더블 기기를 검증할 때 화면 폭만 바꾸면 충분할 줄 알았다. 실제로는 힌지 영역 때문에 보일러 카드의 버튼이 접히는 경계에 걸렸다. 해결책은 `MediaQuery.displayFeatures`를 위젯 곳곳에서 직접 읽지 않고, 앱이 쓸 수 있는 영역을 계산하는 작은 레이어로 분리하는 것이었다. 그러면 일반 태블릿, 펼친 폴더블, 반쯤 접힌 상태를 같은 테스트 계약으로 확인할 수 있다.

## 폭만 보고 배치했을 때 생긴 문제

기존 대시보드는 `width >= 600`이면 2열 Grid를 만들었다. 이 기준은 태블릿에서는 맞았지만, 세로로 접힌 폴더블에서는 힌지를 가로지르는 카드가 생겼다. 화면에 그려지기는 해서 golden test도 통과했다. 손가락이 힌지에 닿는 순간 제어 버튼을 누르기 어려운, 전형적인 “테스트는 초록색인데 사용성은 실패”인 상황이었다.

| 환경 | 폭 기준만 적용 | DisplayFeature 반영 |
|---|---|---|
| 일반 휴대폰 | 1열 | 1열 |
| 태블릿 | 2열 | 2열 |
| 세로 폴더블 | 힌지 위에 카드 배치 | 좌우 패널 분리 |
| 반쯤 접힌 기기 | 버튼이 접힘 경계에 위치 | 한쪽 패널에 제어 UI 고정 |

## 레이아웃 정보를 앱 규칙으로 감싼다

아래 객체는 Flutter의 플랫폼 정보와 화면 배치 규칙 사이에 두는 경계다.

```dart
enum PanelLayout { single, split }

class WindowLayoutInfo {
  const WindowLayoutInfo({required this.width, this.hasHinge = false});

  final double width;
  final bool hasHinge;

  PanelLayout get panelLayout =>
      hasHinge || width >= 600 ? PanelLayout.split : PanelLayout.single;
}

WindowLayoutInfo readWindowLayout(BuildContext context) {
  final media = MediaQuery.of(context);
  final hasHinge = media.displayFeatures.any(
    (feature) => feature.type == DisplayFeatureType.hinge,
  );
  return WindowLayoutInfo(width: media.size.width, hasHinge: hasHinge);
}
```

`DeviceDashboard`는 `WindowLayoutInfo`만 받게 했다. 그래서 실제 앱에서는 `readWindowLayout(context)`를 넘기고, 테스트에서는 힌지가 있다고 명시할 수 있다. `displayFeatures`가 비어 있다는 이유로 폴더블 분기 자체를 테스트하지 못하는 일이 사라진다.

## integration_test에서 두 패널을 확인한다

화면 폭을 억지로 조작하는 대신 테스트 빌더에서 레이아웃 정보를 주입하면 힌지 경계에 제어 버튼이 놓이지 않는 조건을 검증할 수 있다.

```dart
testWidgets('힌지가 있으면 제어 패널을 분리한다', (tester) async {
  await tester.pumpWidget(TestApp(
    layout: const WindowLayoutInfo(width: 673, hasHinge: true),
  ));

  expect(find.byKey(const Key('device-panel-left')), findsOneWidget);
  expect(find.byKey(const Key('device-panel-right')), findsOneWidget);
  expect(find.byKey(const Key('boiler-power-button')), findsOneWidget);
});
```

실제 기기 테스트에서는 Android 폴더블 에뮬레이터의 접힘 상태도 별도 smoke 테스트로 남겼다. 여기서 모든 화면을 다 돌리려고 하면 실행 시간이 급격히 늘어난다. 대시보드, 기기 상세, 설정처럼 패널 분기가 있는 세 화면만 고정하고 나머지는 `WindowLayoutInfo` 단위 테스트로 보냈다.

## 주의할 점

`DisplayFeatureType.hinge`만 검사하면 모든 접힘 상태를 설명할 수 있는 것은 아니다. 제조사와 OS에 따라 `fold`나 `cutout`이 보고될 수 있으므로, 제품에서 지원할 범위를 먼저 정해야 한다. 또 힌지가 있다고 무조건 2열로 만들면 폭이 좁은 반쪽 패널에서 텍스트가 다시 넘친다. 각 패널의 최소 폭과 터치 영역 48dp를 함께 확인해야 한다.

짧게 정리하면, Flutter 폴더블 대응은 `width` 조건문을 늘리는 일이 아니다. `DisplayFeature`를 레이아웃 입력으로 추상화하고, integration_test에서는 힌지 유무·패널 수·핵심 제어 버튼 위치를 계약으로 고정해야 한다. 그래야 IoT 화면이 펼친 상태에서만 그럴듯한 UI로 남지 않는다.
