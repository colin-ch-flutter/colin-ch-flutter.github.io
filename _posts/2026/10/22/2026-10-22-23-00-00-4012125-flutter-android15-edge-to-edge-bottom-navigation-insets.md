---
layout: post
title: "Flutter Android 15·16 edge-to-edge 오류 - BottomNavigation이 시스템 바에 가릴 때"
description: "Flutter 3.27 이후 Android 15·16 release에서 BottomNavigation과 하단 버튼이 시스템 내비게이션 바에 가려질 때 viewPadding, SafeArea, 임시 opt-out을 선택하는 기준을 정리한다."
date: 2026-10-22
tags: [Flutter, Android, 배포·운영, UI, 성능최적화]
comments: true
share: true
---

![Flutter Android edge-to-edge 인셋 처리](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

Android 15 이상을 대상으로 Flutter 앱을 release하면 하단 버튼이나 `BottomNavigationBar`가 시스템 내비게이션 바 아래로 들어갈 수 있다. 앱을 새로 출시하는 경우엔 edge-to-edge를 유지하고 `SafeArea` 또는 `viewPadding`으로 콘텐츠를 안쪽으로 밀어야 한다. Android 15에서만 급히 기존 화면을 살려야 할 때는 opt-out을 검토할 수 있지만, Android 16에서는 이 우회 설정을 계속 기대하면 안 된다. 이 글의 기준은 Flutter 공식 문서가 2026년 9월 21일 기준 Flutter 3.47 문서로 표시한 내용이다.

그림에서 볼 부분은 앱 화면의 높이가 시스템 바를 제외한 값이 아니라, 시스템 바 영역까지 포함할 수 있다는 점이다.

## 왜 release에서만 하단이 가려지나

Flutter 3.27부터 기본 target SDK가 Android 15(API 35)로 올라갔다. Android 15는 target SDK 35 앱에 edge-to-edge를 강제하므로 상태 바와 내비게이션 바가 Flutter 화면 위에 겹칠 수 있다. 디버그에서 보이지 않던 문제가 Play 출시용 target SDK나 특정 제조사 설정에서 드러나는 이유다.

| 확인할 조건 | 판단 |
|---|---|
| Android 15(API 35) 이상 | 시스템 바 영역을 직접 고려해야 한다 |
| Android 16(API 36) 이상 | edge-to-edge opt-out을 사용할 수 없다 |
| 하단 고정 UI | `SafeArea` 또는 `viewPadding.bottom` 확인 |
| 키보드가 함께 열리는 화면 | `padding.bottom`과 `viewPadding.bottom`의 차이를 확인 |

Android 15에서 콘텐츠가 가려지는 현상은 Flutter 위젯 자체의 높이 계산 오류라기보다, 이전에 시스템이 제공하던 바깥 여백이 사라진 결과인 경우가 많다.

## 하단 내비게이션은 SafeArea부터 확인한다

하단 고정 메뉴를 시스템 바 위에 놓는 목적이라면 아래처럼 `SafeArea`를 적용하는 것이 가장 단순하다. 화면 전체를 edge-to-edge로 그리고, 메뉴만 안전 영역을 지키는 방식이다.

```dart
Scaffold(
  body: const HomePage(),
  bottomNavigationBar: SafeArea(
    top: false,
    child: NavigationBar(
      selectedIndex: 0,
      onDestinationSelected: _selectTab,
      destinations: const [
        NavigationDestination(icon: Icon(Icons.home), label: '홈'),
        NavigationDestination(icon: Icon(Icons.settings), label: '설정'),
      ],
    ),
  ),
);
```

직접 만든 하단 컨테이너라면 `MediaQuery.viewPaddingOf(context).bottom`을 기준으로 삼는다. `viewPadding`은 키보드가 열려도 시스템 바의 원래 안전 영역을 나타내므로, 키보드가 열리고 닫힐 때 메뉴가 갑자기 점프하는 것을 줄이는 데 유리하다.

```dart
final bottomInset = MediaQuery.viewPaddingOf(context).bottom;

Padding(
  padding: EdgeInsets.only(bottom: bottomInset),
  child: const CustomBottomBar(),
)
```

반대로 입력창 바로 위에 붙어야 하는 영역은 `MediaQuery.viewInsetsOf(context).bottom`이 필요하다. 시스템 바용 여백과 키보드용 여백을 같은 값으로 처리하면 키보드가 열린 상태에서 빈 공간이 두 배가 되거나 버튼이 밀릴 수 있다.

## Android 15에서만 임시로 되돌리는 경우

기존 화면을 짧은 기간 유지해야 한다면 Android 공식 마이그레이션 방식대로 `android/app/src/main/res/values/styles.xml`과 `values-night/styles.xml`의 `LaunchTheme`, `NormalTheme`에 아래 속성을 넣을 수 있다.

```xml
<item name="android:windowOptOutEdgeToEdgeEnforcement">true</item>
```

다만 이 설정은 Android 15용 임시 대응이다. Android 16에서는 해당 opt-out 속성이 비활성화되며, 잘못된 리소스 구성을 유지하면 실행 문제가 생길 수 있다. 새 화면을 만드는 프로젝트라면 이 설정으로 시간을 벌기보다 인셋을 적용하는 쪽이 안전하다.

## 출시 전 체크리스트

- [ ] `compileSdk`와 `targetSdk`가 실제 release 빌드에서 몇인지 확인했다.
- [ ] 제스처 내비게이션과 3버튼 내비게이션을 각각 확인했다.
- [ ] Android 15(API 35)와 Android 16(API 36)에서 하단 버튼을 눌러봤다.
- [ ] 키보드를 열었을 때 하단 UI가 두 번 밀리지 않는지 확인했다.
- [ ] 임시 opt-out을 썼다면 `values-35` 등 버전별 리소스 전략을 검토했다.

핵심은 `SystemChrome` 호출로 시스템 바 색상만 바꾸는 것이 해결책이 아니라는 점이다. 가려진 콘텐츠는 `SafeArea`나 정확한 `MediaQuery` 인셋으로 고치고, opt-out은 Android 15의 출시 지연을 줄이는 임시 선택으로만 남겨야 한다.

참고: [Flutter edge-to-edge 변경 안내](https://docs.flutter.dev/release/breaking-changes/default-systemuimode-edge-to-edge), [Android 15 동작 변경](https://developer.android.com/about/versions/15/behavior-changes-15), [Android 16 동작 변경](https://developer.android.com/about/versions/16/behavior-changes-16)
