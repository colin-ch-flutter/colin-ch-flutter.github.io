---
layout: post
title: "Flutter Android 16 예측 뒤로가기 오류 - onBackPressed 대신 PopScope로 출시 대응"
description: "Flutter 앱이 Android 16에서 뒤로가기 콜백을 놓치는 이유와 target SDK 36, AndroidManifest, PopScope를 출시 빌드에 적용하는 순서를 정리한다."
date: 2026-10-01
tags: [Flutter, Android, 배포, 테스트, UI]
comments: true
share: true
---
![Flutter Android 16 예측 뒤로가기 출시 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Android 16(API 36)을 대상으로 빌드할 앱이라면 `onBackPressed`나 Flutter의 `WillPopScope`에 출시 동작을 맡기면 안 된다. 뒤로가기를 막거나 저장 확인을 해야 한다면 `PopScope`로 바꾸고, 단순히 `Navigator.pop`만 호출하는 화면은 별도 차단 로직을 추가하지 않는다.

## 증상이 생기는 조건

target SDK 36 이상이면서 Android 16 이상이면 시스템 예측 뒤로가기가 기본 활성화된다. 이때 `onBackPressed`는 호출되지 않고 `KEYCODE_BACK`도 전달되지 않는다. Flutter 3.22부터는 `WillPopScope` 대신 미리 `canPop` 상태를 정하는 `PopScope`가 권장된다. 저장할 내용이 없는 상세 화면은 별도 차단 로직 없이 기본 Navigator 동작을 유지하면 된다.

## `WillPopScope`를 `PopScope`로 바꾸기

저장되지 않은 폼을 닫기 전에 확인 대화상자를 띄우는 경우처럼, 뒤로가기를 조건부로 막아야 하는 코드다.

```dart
return PopScope<void>(
  canPop: false,
  onPopInvokedWithResult: (didPop, result) async {
    if (didPop || !context.mounted) return;

    final leave = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('작성 내용을 버릴까?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('취소'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('나가기'),
          ),
        ],
      ),
    );

    if (leave == true && context.mounted) {
      Navigator.of(context).pop();
    }
  },
  child: const EditPageBody(),
);
```

`canPop: false`는 제스처 시작 시점에 화면을 닫지 않겠다는 뜻이다. `didPop`이 `true`라면 다시 `pop()`하지 않아야 이중 이동을 막을 수 있다.

## Android 설정과 출시 체크

예측 뒤로가기 설정은 앱의 `<application>`에 둔다.

```xml
<application
    android:name="${applicationName}"
    android:label="my_app"
    android:enableOnBackInvokedCallback="true">
</application>
```

출시 전 확인할 항목은 세 가지다.

- `rg "WillPopScope|onBackPressed|KEYCODE_BACK" lib android`로 기존 차단 코드를 찾는다.
- `flutter build appbundle --release`로 target SDK 36 산출물을 만든다.
- Android 16 에뮬레이터에서 제스처·3버튼 뒤로가기와 대화상자 취소·확정을 확인한다.

아직 `PopScope`로 옮길 수 없는 네이티브 화면은 `android:enableOnBackInvokedCallback="false"`로 임시 회피할 수 있다. 예측 뒤로가기 애니메이션을 포기하는 호환 설정이므로 장기 해결책으로 쓰지 않는다.

`targetSdkVersion 36 + Android 16` 조합에서는 예전 뒤로가기 콜백을 믿지 않는다. 차단 조건은 `PopScope.canPop`, 처리 결과는 `didPop`으로 확인하고 출시 전 제스처와 버튼을 따로 실행한다.

근거: [Flutter 예측 뒤로가기 마이그레이션](https://docs.flutter.dev/release/breaking-changes/android-predictive-back), [Flutter Android 예측 뒤로가기 설정](https://docs.flutter.dev/platform-integration/android/predictive-back), [Android 16 동작 변경](https://developer.android.com/about/versions/16/behavior-changes-16)
