---
layout: post
title: "Flutter Android 15 edge-to-edge 레이아웃 오류 - SafeArea와 출시 체크리스트"
description: "Flutter 3.27 이후 Android 15 출시 빌드에서 화면이 시스템 바 아래로 깔리는 이유와 SafeArea, viewPadding, targetSdk 조건별 복구 방법을 정리한다."
date: 2026-09-20
tags: [Flutter, Android, 성능최적화, 배포·운영]
comments: true
share: true
---

![Android 15 edge-to-edge 앱 화면과 시스템 영역](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Android 15 이상에서 Flutter 앱의 버튼이나 하단 탭이 내비게이션 바 아래로 깔진다면 `SafeArea`를 무작정 늘리기보다 `targetSdkVersion`과 화면의 책임 범위를 나눠야 한다. 일반적인 폼·목록 앱은 `Scaffold`의 본문을 `SafeArea`로 감싸는 선택이 가장 빠르다. 반대로 지도·게임처럼 화면 전체를 그려야 하는 앱은 시스템 인셋을 직접 읽어 HUD와 콘텐츠에 다르게 적용해야 한다.

## 왜 출시 빌드에서만 화면이 달라지나

Android 15(API 35)는 edge-to-edge가 기본이다. Flutter 3.27부터 기본 target SDK가 Android 15가 되므로, SDK를 직접 35로 올리지 않았어도 새 Flutter 프로젝트나 업그레이드한 프로젝트가 영향을 받을 수 있다. 상태 표시줄은 투명해지고, 콘텐츠는 시스템 바 뒤까지 그려진다.

Flutter 공식 문서 기준으로 Android 16부터는 edge-to-edge 해제를 전제로 한 우회 설정도 안전하지 않다. `android:windowOptOutEdgeToEdgeEnforcement`는 Android 15에서 시간을 버는 용도이지 장기 해결책이 아니다. 출처와 확인 날짜는 2026-09-20 기준이다.

| 화면 유형 | 권장 처리 | 피할 처리 |
|---|---|---|
| 일반 목록·폼 | `Scaffold` 본문에 `SafeArea` 적용 | 모든 위젯에 임의의 `padding: 24` 추가 |
| 하단 탭·CTA | `MediaQuery.viewPadding.bottom`을 하단 여백에 반영 | 고정 높이만 믿고 하단 여백 생략 |
| 게임·지도·카메라 | 전체 화면 유지 후 HUD만 인셋 적용 | 전체 화면을 `SafeArea`로 감싸 콘텐츠 축소 |

## 일반 앱의 최소 수정

화면 전체가 시스템 바를 침범하지 않아야 하는 경우 아래처럼 경계를 한 곳에 둔다. 위젯마다 SafeArea를 중첩하면 기기별로 여백이 두 번 적용될 수 있다.

```dart
class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('홈')),
      body: SafeArea(
        child: ListView(
          padding: const EdgeInsets.fromLTRB(16, 12, 16, 24),
          children: const [
            Text('콘텐츠'),
          ],
        ),
      ),
    );
  }
}
```

`AppBar`와 `BottomNavigationBar`의 동작까지 한 번에 바꾸려 하지 않는 것도 중요하다. 본문이 가려지는지, 하단 액션만 가려지는지 분리해서 확인해야 한다. Flutter의 `SafeArea`는 `MediaQuery.padding`을 소비하므로, 그 안쪽 위젯에서 같은 값을 다시 더하면 여백이 커진다.

## 하단 고정 버튼은 `viewPadding`을 읽는다

키보드가 올라왔을 때의 `viewInsets`와 항상 존재하는 시스템 영역의 `viewPadding`은 용도가 다르다. 출시 화면의 하단 버튼이 제스처 내비게이션 영역과 겹치는 문제에는 `viewPadding.bottom`을 사용한다.

```dart
class BottomAction extends StatelessWidget {
  const BottomAction({super.key, required this.onPressed});

  final VoidCallback onPressed;

  @override
  Widget build(BuildContext context) {
    final bottom = MediaQuery.viewPaddingOf(context).bottom;

    return Padding(
      padding: EdgeInsets.fromLTRB(16, 8, 16, 12 + bottom),
      child: SizedBox(
        width: double.infinity,
        child: FilledButton(
          onPressed: onPressed,
          child: const Text('저장'),
        ),
      ),
    );
  }
}
```

키보드가 열린 상태에서 버튼을 키보드 위로 올려야 한다면 `MediaQuery.viewInsetsOf(context).bottom`을 별도로 고려한다. 두 값을 무조건 더하면 키보드가 닫힌 기기에서 여백이 과해질 수 있으므로, 실제 화면 정책에 맞춰 `max(viewPadding.bottom, viewInsets.bottom)`처럼 선택한다.

## 출시 전 복구 체크리스트

- [ ] `flutter --version`과 `flutter.targetSdkVersion`을 기록했다.
- [ ] Android 15(API 35) 에뮬레이터에서 3버튼·제스처 내비게이션을 각각 확인했다.
- [ ] 상태 표시줄이 투명한 화면에서 AppBar 제목과 입력창이 가려지지 않는다.
- [ ] 하단 버튼·탭이 제스처 영역과 겹치지 않는다.
- [ ] 키보드를 열고 닫아도 하단 여백이 두 배가 되지 않는다.
- [ ] `values/`와 `values-night/` 스타일을 수정했다면 두 파일의 정책이 다르지 않다.
- [ ] Android 16 대상에서는 opt-out 속성을 장기 해법으로 남기지 않았다.

검증이 급하면 `flutter build appbundle --release`만 통과시키지 말고 API 35 에뮬레이터에서 실제 레이아웃을 캡처해야 한다. 이 문제는 컴파일 오류가 아니라 시스템 인셋 계산의 변화라서 빌드 성공만으로 해결 여부를 판단할 수 없다.

Flutter의 [Android 15 edge-to-edge 마이그레이션 문서](https://docs.flutter.dev/release/breaking-changes/default-systemuimode-edge-to-edge)는 target SDK 35부터의 기본 동작과 Android 16의 제한을 설명한다. Android의 [공식 동작 변경 문서](https://developer.android.com/about/versions/15/behavior-changes-15)도 시스템 바 뒤로 콘텐츠가 그려질 수 있음을 명시한다. 앱의 목적이 전체 화면이 아니라면 구매할 패키지나 네이티브 플러그인 없이 `SafeArea`와 인셋 분리만으로 대응할 수 있다.
