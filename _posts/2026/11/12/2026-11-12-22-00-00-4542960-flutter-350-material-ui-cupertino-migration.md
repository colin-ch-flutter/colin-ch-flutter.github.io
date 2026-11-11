---
layout: post
title: "Flutter 3.50 Material UI 오류 - 독립 패키지 전환 뒤 타입 충돌 줄이기"
description: "Flutter 3.50에서 material_ui·cupertino_ui로 전환할 때 dart fix만 믿지 않고 의존 패키지 타입 충돌과 로컬라이제이션을 출시 전에 점검하는 방법을 정리한다."
date: 2026-11-12
tags: [Flutter, Dart, Material 3, Cupertino, 출시]
comments: true
share: true
---
![Flutter 3.50 Material UI와 Cupertino UI 독립 패키지 전환 점검](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

Flutter 3.50으로 올린 뒤 `package:flutter/material.dart`와 `package:flutter/cupertino.dart` 경고가 늘었다면, 앱 규모가 작을 때는 `material_ui`·`cupertino_ui`로 바로 전환하는 편이 낫다. 다만 공개 패키지가 `ThemeData`, `ColorScheme` 같은 타입을 예전 라이브러리에서 노출한다면 자동 수정만으로는 끝나지 않는다. 그런 앱은 의존성 업데이트 가능 여부를 먼저 확인하고, 전환을 미루는 선택도 유효하다.

## 무엇이 바뀌었나

Flutter 공식 문서 기준으로 독립 패키지는 Flutter 3.47부터 사용할 수 있었고, 3.50에서 기존 프레임워크 내 Material·Cupertino 라이브러리가 정식 deprecated 대상이 됐다. 위젯 이름이 바뀐 게 아니라 import의 소유자가 바뀐 것이다.

| 확인 대상 | 기존 | 전환 후 |
| --- | --- | --- |
| Material 위젯 | `package:flutter/material.dart` | `package:material_ui/material_ui.dart` |
| Cupertino 위젯 | `package:flutter/cupertino.dart` | `package:cupertino_ui/cupertino_ui.dart` |
| 전환 도구 | 수동 import 수정 | `dart fix --apply --code=migrate_design_widgets` |
| 위험 지점 | SDK 타입 공유 | 서로 다른 패키지 타입 간 불일치 |

앱 코드만 바꾸고 패키지 버전을 고정하면 `ThemeData`나 `TextTheme`를 함수 인자로 주고받는 지점에서 컴파일 오류가 날 수 있다. 이름이 같아도 Dart에서는 다른 라이브러리의 타입이다.

## 안전한 전환 순서

코드를 일괄 치환하기 전에 현재 Flutter 버전과 의존 패키지를 기록한다.

```bash
flutter --version
flutter pub outdated
rg "package:flutter/(material|cupertino)\.dart" lib test packages -g '*.dart'
```

앱이 독립 패키지를 사용할 수 있는 상태라면 공식 자동 수정으로 import와 의존성을 바꾼다.

```bash
dart fix --apply --code=migrate_design_widgets
flutter pub add material_ui
flutter pub add cupertino_ui
flutter pub get
```

실제 앱에서 Material만 쓴다면 `cupertino_ui`를 억지로 추가할 필요는 없다. 반대로 `flutter_localizations`를 직접 구성했다면 로컬라이제이션도 확인해야 한다. 독립 Material 패키지에서는 아래처럼 delegate 묶음을 사용할 수 있다.

```dart
import 'package:material_ui/material_ui.dart';

MaterialApp(
  localizationsDelegates: GlobalMaterialLocalizations.delegates,
  supportedLocales: const [Locale('ko'), Locale('en')],
)
```

이 코드는 기존 `GlobalMaterialLocalizations`, `GlobalCupertinoLocalizations`, `GlobalWidgetsLocalizations`를 각각 나열하던 구성을 줄이는 예다. 프로젝트의 실제 지원 언어 목록은 유지해야 한다.

## 자동 수정 뒤 남는 오류

서드파티 패키지가 예전 import를 계속 사용하면 `MaterialUiCompatibilityBridge`가 임시 다리가 될 수 있다. 하지만 패키지 API가 예전 `ColorScheme`나 `TextTheme`를 public 타입으로 노출한다면 bridge만으로 해결되지 않는다. 이때는 해당 패키지의 독립 패키지 대응 버전으로 올리거나, 전환 전까지 앱 import를 유지해야 한다.

출시 전에는 아래 세 가지를 분리해서 확인한다.

- `rg "package:flutter/(material|cupertino)\.dart" lib test` 결과가 앱의 의도된 호환 코드만 남는가
- `flutter analyze`에서 `ThemeData`, `ColorScheme`, `TextTheme` 타입 충돌이 없는가
- `flutter build appbundle --release` 또는 `flutter build ipa --release`가 새 의존성 잠금 파일로 통과하는가

`dart fix`가 성공했다는 메시지는 패키지 생태계 전체가 전환됐다는 뜻이 아니다. 특히 디자인 위젯 타입을 외부 API에 전달하는 패키지는 major 버전 업데이트와 함께 확인해야 한다. 독립 패키지 전환이 당장 필요하지 않은 레거시 앱이라면 Flutter SDK를 무리하게 올리기보다, 의존 패키지의 대응 릴리스를 기다리는 편이 복구 비용이 작다.

짧게 정리하면, Flutter 3.50 전환은 import 치환보다 타입 경계를 점검하는 작업이다. 앱 코드가 단순하면 `dart fix`로 빠르게 옮기고, 공개 패키지 타입을 공유한다면 호환 버전·bridge·전환 보류를 비교한 뒤 release 빌드까지 확인해야 한다. 기준은 2026년 11월 12일 Flutter 공식 마이그레이션 문서와 `material_ui` 패키지 안내를 확인했다.

참고 문서: [Flutter 독립 Material·Cupertino 패키지 전환](https://docs.flutter.dev/release/breaking-changes/material-ui-and-cupertino-ui), [Flutter 3.50 변경 목록](https://docs.flutter.dev/release/breaking-changes), [material_ui 패키지](https://pub.dev/packages/material_ui)
