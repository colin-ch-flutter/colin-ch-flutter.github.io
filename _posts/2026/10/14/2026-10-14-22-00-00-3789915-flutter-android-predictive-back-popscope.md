---
layout: post
title: "Flutter Android predictive back 오류 - WillPopScope를 PopScope로 바꾸는 출시 조건"
description: "Flutter 앱에서 Android 14 predictive back이 동작하지 않을 때 WillPopScope를 PopScope와 NavigatorPopHandler로 바꾸는 조건, 확인 다이얼로그와 nested Navigator의 차이를 정리한다."
date: 2026-10-14
tags: [Flutter, Android, 테스트, UI]
comments: true
share: true
---

![Flutter Android predictive back 제스처를 점검하는 스마트폰 화면](https://images.unsplash.com/photo-1511707171634-5f897ff02aa9?auto=format&fit=crop&w=1600&q=80)

Android 14 이상에서 뒤로 스와이프 미리보기가 필요하거나 `WillPopScope` deprecated 경고를 만난 앱이라면 `PopScope`로 옮겨야 한다. 단일 화면은 `canPop`을 미리 계산하면 되지만, 중첩 `Navigator`와 확인창은 단순 치환으로 해결되지 않는다.

## 왜 release에서 뒤로 가기 동작이 달라지는가

Android 14(API 34)는 predictive back을 지원한다. Flutter 공식 문서 기준으로 제스처가 끝나는 시점에 취소 여부를 정하는 `WillPopScope`는 미리보기와 맞지 않는다. 앱은 화면이 빌드될 때 pop 가능 여부를 알려야 한다.

Flutter 프로젝트의 Android manifest도 확인해야 한다. Flutter 공식 안내는 Android 13 이상에서 `<application>`에 `android:enableOnBackInvokedCallback="true"`를 선언하도록 한다. 이 값이 빠져 있으면 Dart 코드를 바꿔도 기기 설정과 OS 버전에 따라 예전 back 처리처럼 보일 수 있다.

```xml
<application
    android:name="${applicationName}"
    android:label="my_app"
    android:icon="@mipmap/ic_launcher"
    android:enableOnBackInvokedCallback="true">
```

그림에서 볼 부분은 시스템이 뒤 화면을 미리 보여주려면 pop 가능 여부를 사전에 알아야 한다는 점이다.

## 단일 화면은 `canPop`을 상태로 만든다

편집 화면에서 입력값이 바뀌었을 때만 뒤로 가기를 막는 예시는 아래와 같다. `onWillPop` 안에서 판단하지 않고 변경 여부를 상태로 유지한다.

```dart
return PopScope<void>(
  canPop: !_dirty,
  onPopInvokedWithResult: (didPop, result) {
    if (!didPop) {
      debugPrint('저장하지 않은 편집 내용이 남아 있다.');
    }
  },
  child: Scaffold(
    appBar: AppBar(title: const Text('편집')),
    body: TextField(
      onChanged: (_) => setState(() => _dirty = true),
    ),
  ),
);
```

확인창이 필요하면 `canPop: false`로 막은 뒤 `didPop == false`일 때만 창을 열고, 버리기를 선택하면 `Navigator.pop()`을 직접 호출한다. `_dialogOpen` 같은 잠금 상태로 중복 창도 막는다.

## nested Navigator는 `PopScope` 하나로 감싸지 않는다

하단 탭처럼 shell route 안에 별도 `Navigator`가 있다면, 안쪽 스택보다 바깥 route가 먼저 닫힐 수 있다. 이 경우 Flutter 공식 문서의 `NavigatorPopHandler`로 책임을 분리한다.

```dart
NavigatorPopHandler(
  onPop: () => _nestedNavigatorKey.currentState?.pop(),
  child: Navigator(
    key: _nestedNavigatorKey,
    onGenerateRoute: _onGenerateRoute,
  ),
)
```

여러 `PopScope`를 같은 `ModalRoute`에 겹쳐 두면 우선순위가 애매해질 수 있다. 화면의 뒤로 가기 책임자를 한 곳으로 모으고 `flutter analyze`로 SDK 시그니처를 확인한다.

## 출시 전 확인 항목

- [ ] `rg "WillPopScope|Navigator\.willPop" lib` 결과가 0건이다.
- [ ] Android 14 이상에서 제스처 시작 시 이전 route 미리보기가 보인다.
- [ ] `canPop`이 입력 변경과 같은 프레임에 갱신된다.
- [ ] 확인 다이얼로그에서 취소·버리기·저장을 각각 한 번씩 검사했다.
- [ ] nested Navigator가 있으면 안쪽 route, 바깥 route 순서로 pop된다.
- [ ] Android 13의 하드웨어 back도 별도로 눌러 봤다.
- [ ] `flutter analyze`와 release 빌드에서 deprecated 경고·동작을 확인했다.

Android 13 이하에서도 앱은 동작할 수 있지만 predictive back 미리보기가 같다고 가정하면 안 된다. `canPop`을 항상 `false`로 두면 정상 이동까지 막히며, `onPopInvokedWithResult`는 pop 전에 승인하는 콜백이 아니라 처리 결과를 알리는 콜백이다.

단일 route는 `PopScope.canPop`을 미리 계산하고, 중첩 navigator는 `NavigatorPopHandler`로 안쪽 스택을 먼저 처리해야 한다. Android 14 대응의 핵심은 뒤로 가기 가능 상태를 빌드 시점에 노출하는 것이다.

참고 문서: [Flutter predictive back 추가 안내](https://docs.flutter.dev/platform-integration/android/predictive-back), [Flutter Android predictive back 변경 안내](https://docs.flutter.dev/release/breaking-changes/android-predictive-back), [Android Developers predictive back](https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture), [Flutter PopScope API](https://api.flutter.dev/flutter/widgets/PopScope-class.html)
