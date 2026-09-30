---
layout: post
title: "Flutter Android 17 화면 회전 오류 - 태블릿에서 setPreferredOrientations가 무시될 때"
description: "Flutter Android 17에서 태블릿·폴더블의 화면 방향 잠금이 무시되는 조건과 600dp 기준의 적응형 레이아웃 전환 방법을 정리한다."
date: 2026-09-30
tags: [Flutter, Android, 반응형, 배포·운영, UI]
comments: true
share: true
---

![Flutter Android 17 태블릿과 폴더블 화면 회전 대응](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

이 그림에서는 한 방향으로 고정된 화면이 아니라 태블릿과 폴더블의 가용 폭에 맞춰 바뀌는 UI를 봐야 한다.

Android 17(API 37) 대상으로 출시를 준비하는 Flutter 앱이라면 태블릿·폴더블에서 `SystemChrome.setPreferredOrientations`가 무시될 수 있다. 휴대폰만 지원하는 앱은 기존 잠금을 유지해도 되지만, `smallest width 600dp` 이상 화면까지 배포한다면 회전을 막는 설정을 고치는 것보다 레이아웃을 회전 가능한 구조로 바꾸는 선택이 안전하다.

## 어떤 앱이 영향을 받나

Flutter 공식 breaking change 문서 기준으로 Android 17 이상을 타깃하면 600dp 이상 대형 화면에서 `SystemChrome.setPreferredOrientations`가 무시된다. 매니페스트의 `screenOrientation`, `resizeableActivity`, 종횡비 제한도 같은 조건에서 효력이 사라진다. Android 16에서는 임시 opt-out이 있었지만 Android 17에서는 제거된다.

| 환경 | 화면 방향 잠금 | 대응 |
|---|---:|---|
| 일반 휴대폰, 600dp 미만 | 유지될 수 있음 | 기존 정책 점검 |
| 태블릿·폴더블 펼침, 600dp 이상 | 무시될 수 있음 | 가로·세로 레이아웃 지원 |
| 게임으로 분류된 앱 | 적용 범위가 다름 | 게임 정책과 기기별 테스트 필요 |

문제는 앱이 갑자기 회전하는 것 자체가 아니다. 회전 뒤 Activity가 재생성되면서 입력 중인 폼, 선택된 탭, 스크롤 위치가 사라지거나 고정 폭 위젯이 깨지는 것이 실제 출시 장애다.

## 회전 잠금 코드를 조건부로 줄인다

화면 폭이 작은 기기에서만 세로를 유지하고 대형 화면에서는 잠금을 해제하려면 Flutter의 화면 크기 정보를 기준으로 호출한다. 이 코드는 Android 17의 시스템 동작을 되돌리는 우회가 아니라, 작은 화면과 큰 화면의 정책을 분리하는 예시다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

Future<void> updateOrientationPolicy(BuildContext context) async {
  final width = MediaQuery.sizeOf(context).shortestSide;

  if (width < 600) {
    await SystemChrome.setPreferredOrientations(const [
      DeviceOrientation.portraitUp,
    ]);
  } else {
    await SystemChrome.setPreferredOrientations(const []);
  }
}
```

다만 `MediaQuery` 값은 회전과 창 크기 변경 뒤 달라진다. `build` 안에서 매번 비동기 호출하지 말고, 화면 크기 변경을 감지하는 별도 위젯이나 앱 셸에서 정책을 갱신해야 한다. 화면 방향이 바뀌어도 상태를 보존하도록 폼 값은 `RestorationMixin`, 상태관리 저장소, 또는 명시적인 draft 모델 중 하나에 보관한다.

## 고정 폭 UI를 먼저 찾는다

`Row` 안에 고정된 `SizedBox(width: 400)`를 넣은 화면은 가로·세로 전환 때 바로 overflow가 날 수 있다. 대형 화면에서는 `LayoutBuilder`로 600dp 전후의 배치를 나누고, 좁은 화면에서는 세로 목록을 보여주는 방식이 현실적이다.

```dart
LayoutBuilder(
  builder: (context, constraints) {
    final isLarge = constraints.maxWidth >= 600;
    return isLarge
        ? const TwoPaneSettingsView()
        : const SinglePaneSettingsView();
  },
)
```

출시 전에는 Pixel Tablet과 폴더블 에뮬레이터에서 세 가지를 따로 확인해야 한다.

- 세로에서 입력한 폼을 가로로 돌린 뒤 값과 포커스가 유지되는가
- 가로에서 창을 분할하거나 폭을 줄였을 때 overflow가 없는가
- Android 16과 Android 17 타깃 빌드에서 방향 잠금 결과가 다른가

특히 `flutter run`의 현재 기기 테스트만 통과했다고 Android 17 출시를 판단하면 안 된다. `targetSdk`와 기기 최소 폭 조합이 달라야 이 변경이 나타난다.

## 짧은 점검표

| 점검 | 통과 기준 |
|---|---|
| 방향 API 검색 | `setPreferredOrientations` 사용 위치를 확인했다 |
| 대형 화면 | smallest width 600dp 이상에서 가로·세로가 모두 열린다 |
| 상태 복구 | 회전 뒤 입력·탭·스크롤 상태가 보존된다 |
| 레이아웃 | 고정 폭 대신 `LayoutBuilder` 또는 유연한 위젯을 쓴다 |
| 출시 빌드 | Android 17 타깃 release APK/AAB로 실제 흐름을 재검증했다 |

핵심은 화면 회전 잠금을 끝까지 지키는 것이 아니라, 잠금이 사라져도 사용자가 작업을 잃지 않게 만드는 것이다. Android 17 대형 화면을 지원하지 않는 앱이라면 이 작업을 억지로 추가하기보다 지원 기기 범위와 Play Console의 기기 제외 정책부터 확인하는 편이 비용이 적다.

공식 기준은 [Flutter Android 17 대형 화면 변경 문서](https://docs.flutter.dev/release/breaking-changes/android-large-screens-restrictions-ignored), [Flutter 대형 화면 가이드](https://docs.flutter.dev/ui/adaptive-responsive/large-screens), [Android 적응형 화면 방향·리사이징 문서](https://developer.android.com/develop/adaptive-apps/guides/app-orientation-aspect-ratio-resizability)에서 확인했다.
