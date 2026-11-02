---
layout: post
title: "Flutter Android 17 대화면 오류 - orientation lock이 무시될 때 출시 조건"
description: "Flutter 앱을 targetSdk 37로 올린 뒤 태블릿·폴더블에서 세로 고정이 풀리는 이유와 600dp 기준의 대응, LayoutBuilder 테스트 조건을 정리한다."
date: 2026-11-02
tags: [Flutter, Android, 반응형UI, 태블릿, 배포]
comments: true
share: true
---

![Flutter Android 17 태블릿과 폴더블 대화면 대응](https://images.unsplash.com/photo-1556656793-08538906a9f8?auto=format&fit=crop&w=1600&q=80)

이 그림에서 봐야 할 부분은 휴대폰 화면을 크게 늘리는 것이 아니라, 태블릿·폴더블의 창 너비에 따라 내비게이션과 콘텐츠 구조를 바꾸는 점이다.

`targetSdk 37`로 Android 17 출시를 준비하면서 `SystemChrome.setPreferredOrientations`가 더 이상 태블릿과 폴더블의 세로 고정을 보장하지 않는다면, 설정을 되살리는 것보다 화면을 적응형으로 바꾸는 편이 맞다. Android 17은 최소 너비 600dp 이상에서 방향·리사이즈·가로세로 비율 제한을 무시한다. 일반 휴대폰만 지원하고 대화면을 제품 범위에서 제외한 앱에는 이 변경이 바로 적용되지 않는다.

## 문제가 드러나는 조건

| 조건 | 결과 | 출시 판단 |
|---|---|---|
| targetSdk 36 이하 | Android 17에서도 앱별 조건을 따로 확인해야 함 | 호환성 테스트 필요 |
| targetSdk 37 이상 + 600dp 미만 | 휴대폰 방향 고정이 남을 수 있음 | 기존 UX 유지 가능 |
| targetSdk 37 이상 + 600dp 이상 | `setPreferredOrientations`가 무시됨 | 적응형 레이아웃 필수 |
| 폴더블 펼침·분할 화면 | 창 너비가 계속 변함 | 기기 종류보다 현재 너비를 기준으로 판단 |

문제는 `portraitUp` 한 줄이 실패하는 데서 끝나지 않는다. 고정 폭 카드와 하단 버튼이 가로 모드나 분할 창에서 잘릴 수 있다. 경계는 물리 해상도가 아니라 smallest width 600dp다.

## 전역 방향 고정보다 창 너비를 기준으로 바꾼다

아래 코드는 방향을 강제로 잠그는 대신, Flutter가 받은 실제 창 제약으로 내비게이션 형태를 바꾸는 최소 예시다.

```dart
class HomeShell extends StatelessWidget {
  const HomeShell({super.key});

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final isLarge = constraints.maxWidth >= 600;
        return Scaffold(
          body: Row(
            children: [
              if (isLarge) const NavigationRail(selectedIndex: 0),
              const Expanded(child: DashboardPage()),
            ],
          ),
          bottomNavigationBar: isLarge
              ? null
              : const NavigationBar(selectedIndex: 0, destinations: []),
        );
      },
    );
  }
}
```

앱 전체에서 아래처럼 방향을 고정하고 있다면 대화면 대응의 해결책으로 착각하면 안 된다.

```dart
await SystemChrome.setPreferredOrientations([
  DeviceOrientation.portraitUp,
]);
```

휴대폰에서만 세로 UX가 필요하다면 작은 창에서만 이 설정을 적용할 수 있다. 단, 기기 종류를 캐시하지 말고 폴더블 펼침과 분할 화면에서 `LayoutBuilder`가 다시 빌드되는지 확인한다. 대화면 지원 제품은 방향 고정 코드를 제거하고 두 열·NavigationRail 구조를 택하는 편이 낫다.

## 출시 전에 확인할 체크리스트

- [ ] `targetSdk`가 37 이상인지, 600dp 경계에서 어떤 UX를 약속하는지 정했다.
- [ ] `SystemChrome.setPreferredOrientations`와 AndroidManifest의 `screenOrientation`을 모두 검색했다.
- [ ] 600dp 미만 휴대폰, 600dp 이상 태블릿·폴더블을 각각 실행했다.
- [ ] 세로·가로 회전, 분할 화면, 창 크기 변경 뒤 상태와 입력 포커스를 확인했다.
- [ ] `Row`의 고정 폭·`Expanded` 누락·하단 버튼 잘림을 debug와 release에서 확인했다.
- [ ] 방향 고정이 필요한 카메라·게임 화면은 일반 업무 화면과 별도 예외로 문서화했다.

기준은 2026년 11월 2일 확인했다. Flutter 공식 문서는 Android 17 이상에서 600dp 이상의 화면에 `SystemChrome.setPreferredOrientations`가 적용되지 않는다고 안내한다. Android 17에서는 임시 opt-out도 사라졌다.

짧게 정리하면 Android 17 대화면 오류는 Flutter 방향 API의 단순 고장이 아니다. `targetSdk 37`과 600dp 경계를 나누고, 방향이 바뀌어도 깨지지 않는 창 기반 레이아웃을 만든 뒤 태블릿·폴더블 출시 테스트를 통과해야 한다.

참고 문서: [Flutter 대화면 방향·리사이즈 변경](https://docs.flutter.dev/release/breaking-changes/android-large-screens-restrictions-ignored), [Flutter SystemChrome.setPreferredOrientations](https://api.flutter.dev/flutter/services/SystemChrome/setPreferredOrientations.html), [Android 17 동작 변경](https://developer.android.com/about/versions/17/behavior-changes-17)
