---
layout: post
title: "Flutter Android 15·16 edge-to-edge 오류 - 출시 후 상태바와 하단 버튼이 겹칠 때"
description: "Flutter 앱을 Android 15·16과 target SDK 35 이상으로 출시할 때 발생하는 edge-to-edge 겹침을 SafeArea, inset 중복, 임시 opt-out 조건으로 점검하는 방법을 정리한다."
date: 2026-11-16
tags: [Flutter, Android, 배포·운영, UI, 성능최적화]
comments: true
share: true
---
![Flutter Android 15 edge-to-edge 출시 레이아웃 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Flutter 앱을 Android 15·16에 출시한 뒤 상태바 아래로 제목이 들어가거나 하단 버튼이 내비게이션 바에 가려진다면, target SDK를 다시 낮추기보다 edge-to-edge를 기준으로 화면 inset을 정리하는 편이 맞다. 단순 화면이라면 `SafeArea`로 해결할 수 있지만, 이미 `MediaQuery.padding`을 더하는 공통 레이아웃이 있다면 이중 여백이 생긴다.

## 왜 갑자기 겹치는가

Android 문서 기준으로 target SDK 35 이상인 앱은 Android 15 이상에서 화면 전체에 그려진다. 상태바와 내비게이션 바가 차지하던 영역까지 Flutter 캔버스가 사용하므로, 예전처럼 화면 높이를 안전 영역으로 가정한 레이아웃이 그대로 노출된다.

Flutter 공식 문서도 Flutter 3.27부터 기본 `flutter.targetSdkVersion`이 Android 15를 대상으로 하면서 이 변화가 시작된다고 설명한다. Android 16에서는 edge-to-edge를 끄는 기존 우회책을 사용할 때 앱이 충돌할 수 있으므로, `windowOptOutEdgeToEdgeEnforcement`를 영구 해법처럼 넣으면 안 된다.

| 증상 | 실제 원인 | 확인 지점 |
|---|---|---|
| 제목이 상태바와 겹침 | 상단 inset 미반영 | `Scaffold` 안의 공통 AppBar |
| FAB·하단 버튼이 가려짐 | 하단 system bar inset 미반영 | `bottomNavigationBar`, `Stack` |
| 화면마다 여백이 두 배 | `SafeArea`와 수동 padding 중복 | `MediaQuery.paddingOf(context)` |
| Android 16에서 실행 문제 | Android 15용 opt-out 스타일 유지 | `values/`, `values-night/`, `values-35/` |

## Flutter 화면은 안전 영역의 책임을 한 곳에 둔다

화면 전체를 감싸는지, 특정 조작 요소만 감싸는지를 정해야 한다. 아래처럼 화면 전체를 `SafeArea`로 감싸면 일반적인 폼과 목록은 빠르게 안정화할 수 있다.

```dart
class SettingsPage extends StatelessWidget {
  const SettingsPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: ListView(
          padding: const EdgeInsets.all(16),
          children: const [
            Text('설정'),
            SizedBox(height: 24),
            // 설정 항목
          ],
        ),
      ),
    );
  }
}
```

다만 배경 이미지는 상태바 뒤까지 펼치고 버튼만 피해야 한다면 전체 `SafeArea`는 적절하지 않다. 이때는 버튼 영역에만 하단 여백을 적용한다. `SafeArea`와 아래 코드를 같은 위젯 계층에 함께 넣지 않는 것이 핵심이다.

```dart
final bottomInset = MediaQuery.paddingOf(context).bottom;

Positioned(
  left: 16,
  right: 16,
  bottom: bottomInset + 16,
  child: FilledButton(
    onPressed: onSubmit,
    child: const Text('저장'),
  ),
)
```

`Scaffold`의 `bottomNavigationBar`를 사용하는 경우에는 공통 `AppShell`에서 이미 inset을 처리하는지 확인한다. 페이지마다 `SafeArea`를 추가하면 기기와 내비게이션 방식에 따라 하단 간격이 2번 더해질 수 있다.

## 출시 전 점검 순서

1. `android/app/build.gradle` 또는 `.kts`에서 실제 release variant의 `targetSdk`를 확인한다. Flutter 변수에 의존하면 Flutter 버전 변경으로 값이 달라질 수 있다.
2. Android 15/API 35와 Android 16/API 36 에뮬레이터에서 제스처 내비게이션, 3버튼 내비게이션을 각각 확인한다.
3. 세로뿐 아니라 가로 방향에서 display cutout과 하단 버튼을 확인한다.
4. `SafeArea`, `MediaQuery.padding`, `viewPadding`이 한 화면에서 중복 사용되는지 검색한다.
5. release APK 또는 AAB로 확인한다. debug에서만 보이는 시스템 바 설정을 기준으로 판단하지 않는다.

Android 15에 한해 출시를 잠시 미뤄야 한다면 Flutter 문서의 임시 opt-out 스타일을 사용할 수 있다. `values/styles.xml`과 `values-night/styles.xml`에 아래 항목을 넣되, Android 16 이상을 위해 `values-35`에서는 이 항목이 없는 스타일을 제공해야 한다.

```xml
<item name="android:windowOptOutEdgeToEdgeEnforcement">true</item>
```

이 설정은 화면 코드를 고친 것이 아니다. Android 16까지 지원하는 앱이라면 배포 전에 제거하고 inset 기반 레이아웃으로 전환하는 것이 안전하다. 특히 광고 배너, `Stack` 위의 고정 버튼, 커스텀 바텀시트는 `SafeArea` 하나만으로 시각적 겹침과 터치 영역을 모두 해결하지 못할 수 있다.

## 짧은 요점 정리

- target SDK 35 이상과 Android 15 이상 조합에서는 edge-to-edge가 기본이다.
- 전체 화면은 `SafeArea`, 특정 조작 요소는 `MediaQuery.paddingOf` 중 하나로 책임을 정한다.
- Flutter 3.27 이후 target SDK 기본값 변화도 함께 확인한다.
- Android 15 opt-out 스타일은 임시 대응이며 Android 16 출시 전략으로 남겨두지 않는다.

근거: [Flutter edge-to-edge 변경 안내](https://docs.flutter.dev/release/breaking-changes/default-systemuimode-edge-to-edge), [Android edge-to-edge 공식 문서](https://developer.android.com/develop/ui/views/layout/edge-to-edge) (2026-11-16 확인).
