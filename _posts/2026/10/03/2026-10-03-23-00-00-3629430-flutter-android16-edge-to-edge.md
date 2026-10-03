---
layout: post
title: "Flutter Android 16 edge-to-edge 오류 - opt-out 설정으로 release 크래시가 날 때"
description: "Flutter 앱을 targetSdk 36으로 올린 뒤 Android 16에서 화면이 깨지거나 release가 종료될 때, edge-to-edge opt-out 설정을 API 35 리소스로 분리하고 Flutter inset을 점검하는 방법을 정리한다."
date: 2026-10-03
tags: [Flutter, Android, 성능최적화, 배포·운영]
comments: true
share: true
---
![Flutter Android 16 edge-to-edge와 API별 리소스 분기](/images/2026-10-03-flutter-android16-edge-to-edge.png)

Android 16(API 36) 출시를 준비하는 앱은 `windowOptOutEdgeToEdgeEnforcement`를 계속 유지하면 안 된다. API 35에서만 opt-out 스타일을 적용하고, API 36 이상에서는 속성 자체가 없는 리소스를 사용해야 한다. 이미 `targetSdk 36`으로 올렸다면 opt-out을 억지로 유지하기보다 콘텐츠가 상태 표시줄·내비게이션 바 뒤까지 그려지는 edge-to-edge를 기준으로 화면을 고치는 편이 안전하다. Android 14 이하만 지원하거나 targetSdk 35를 당장 유지하는 앱에는 이 글의 리소스 분기가 필수는 아니다.

## 왜 debug는 되고 release에서 문제가 보이나

Android 15에서는 target SDK 35 앱이 edge-to-edge로 동작하지만 `windowOptOutEdgeToEdgeEnforcement=true`로 임시 예외를 둘 수 있다. 문제는 Android 16에서 이 opt-out이 비활성화됐다는 점이다. Flutter 공식 문서도 Android 16 이상에서 같은 메커니즘을 사용하면 앱이 크래시할 수 있다고 안내한다.

여기서 헷갈리기 쉬운 부분은 `compileSdk`, `targetSdk`, 실행 기기의 OS가 서로 다른 값이라는 점이다.

| 조건 | 화면 정책 | 대응 |
|---|---|---|
| target 35 + Android 15 | edge-to-edge 기본, opt-out 가능 | 임시 호환이면 API 35 스타일에만 속성 추가 |
| target 36 + Android 16 | edge-to-edge 강제 | opt-out 속성 제거, `MediaQuery.viewPadding` 등으로 inset 처리 |
| target 36 + Android 14 | 앱은 실행되지만 OS별 차이 존재 | Android 15·16 에뮬레이터에서 회귀 확인 |

## API 35에서만 opt-out 스타일 사용하기

기존 프로젝트가 아직 edge-to-edge 대응을 끝내지 못했고 target 35를 유지해야 한다면, 공통 `values`에 opt-out 스타일을 두고 `values-35`에서 같은 스타일을 덮어쓴다. Android 16은 API 35 리소스를 선택하므로 이 파일에는 문제의 속성을 넣지 않는다.

AndroidManifest에서 Activity가 참조하는 스타일 이름을 확인한다. 보통 Flutter 템플릿은 `@style/LaunchTheme` 또는 `@style/NormalTheme`을 사용한다. 아래 예시는 `NormalTheme`을 기준으로 한 최소 분기다.

`android/app/src/main/res/values/styles.xml`에는 Android 15 이하에서 사용할 임시 opt-out을 둔다.

```xml
<resources>
    <style name="NormalTheme" parent="android:style/Theme.Material.Light.NoActionBar">
        <item name="android:windowOptOutEdgeToEdgeEnforcement">true</item>
        <item name="android:fontFamily">sans</item>
        <item name="android:colorAccent">#2196F3</item>
    </style>
</resources>
```

같은 이름의 `android/app/src/main/res/values-35/styles.xml`을 만들고, API 35 이상에서는 opt-out 속성을 제거한다. 이렇게 해야 Android 16에서 금지된 속성이 선택되지 않는다.

```xml
<resources>
    <style name="NormalTheme" parent="android:style/Theme.Material.Light.NoActionBar">
        <item name="android:fontFamily">sans</item>
        <item name="android:colorAccent">#2196F3</item>
    </style>
</resources>
```

다만 `targetSdk 36`이라면 이 분기를 새 해결책으로 삼으면 안 된다. Android 16에서는 opt-out 자체가 동작하지 않으므로, 앱의 실제 레이아웃을 edge-to-edge에 맞추고 `SafeArea`, `MediaQuery.viewPadding`, 키보드 inset을 점검해야 한다.

## Flutter 화면에서 겹침을 고치는 기준

상단 앱바나 하단 입력창에 고정된 여백을 넣어 둔 코드는 Android 15부터 상태 표시줄과 겹칠 수 있다. Flutter 위젯 전체를 무조건 `SafeArea`로 감싸면 화면 디자인이 바뀌거나 전체 화면 콘텐츠가 잘릴 수 있으므로, 실제로 시스템 바와 만나는 영역만 보호한다.

하단 버튼이 내비게이션 바에 가려지는 화면이라면 아래처럼 `viewPadding.bottom`을 적용한다. 키보드가 열렸을 때는 `viewInsets.bottom`이 별도로 반영되므로 두 값을 무심코 더하지 않는다.

```dart
return Scaffold(
  body: Column(
    children: [
      Expanded(child: content),
      Builder(
        builder: (context) {
          final bottom = MediaQuery.viewPaddingOf(context).bottom;
          return Padding(
            padding: EdgeInsets.only(bottom: bottom + 16),
            child: const SubmitButton(),
          );
        },
      ),
    ],
  ),
);
```

실제 키보드 입력 화면에서는 `Scaffold.resizeToAvoidBottomInset`과 `MediaQuery.viewInsetsOf(context).bottom`의 관계를 확인한다. Android 16 대응을 한다며 모든 숫자를 24나 48로 바꾸는 것은 해결이 아니다. 기기·제스처 내비게이션 설정에 따라 inset 값이 달라지기 때문이다.

## release 전에 확인할 체크리스트

- [ ] `targetSdk`가 35인지 36인지 확인했다.
- [ ] `windowOptOutEdgeToEdgeEnforcement`를 `values-35` 파일에서 제거했다.
- [ ] AndroidManifest가 참조하는 스타일과 리소스 이름이 일치한다.
- [ ] 상태 표시줄과 겹치는 상단 콘텐츠를 API 35·36에서 확인했다.
- [ ] 하단 버튼·입력창·BottomSheet의 `viewPadding`과 `viewInsets`를 따로 확인했다.
- [ ] `flutter build appbundle --release` 산출물을 Android 16 에뮬레이터에서 실행했다.
- [ ] 크래시 로그에 `IllegalStateException` 또는 리소스 테마 관련 오류가 없는지 확인했다.

공식 기준은 2026년 10월 3일 확인했다. Android 16은 target API 36 앱의 edge-to-edge opt-out을 허용하지 않으며, Flutter 문서는 Android 16 이상에서의 크래시를 피하려면 API 35 리소스 분기를 사용하라고 안내한다. 기존 target 35 앱의 임시 호환에는 이 분기가 유효하지만, 새 출시를 target 36으로 맞추는 앱은 edge-to-edge 레이아웃 수정이 최종 해법이다.

참고 문서: [Flutter edge-to-edge 변경 안내](https://docs.flutter.dev/release/breaking-changes/default-systemuimode-edge-to-edge), [Android 16 동작 변경](https://developer.android.com/about/versions/16/behavior-changes-16), [Flutter SystemUiMode API](https://api.flutter.dev/flutter/services/SystemUiMode.html)
