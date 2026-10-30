---
layout: post
title: "Flutter Web 3.50 viewport-fit 오류 - SafeArea가 모바일 노치 아래로 깔릴 때"
description: "Flutter Web 3.50에서 기존 viewport-fit=cover 설정이 실제로 적용되며 노치와 홈 인디케이터 아래에 UI가 놓일 때, SafeArea 적용과 contain 복귀를 선택하는 기준을 정리한다."
date: 2026-10-31
tags: [Flutter, 웹, 접근성, 배포]
comments: true
share: true
---

![모바일 브라우저에서 Flutter Web의 노치와 안전 영역을 점검하는 화면](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

Flutter Web을 3.50으로 올린 뒤 모바일 Safari나 Chrome에서 상단 버튼과 하단 메뉴가 노치·홈 인디케이터에 가려진다면 `web/index.html`의 `viewport-fit=cover`부터 확인해야 한다. 화면을 전체로 채우고 싶은 PWA·키오스크는 `SafeArea`를 필요한 UI에 적용하고, 일반 업무 화면처럼 전체 화면 렌더링이 필요하지 않으면 `viewport-fit=cover`를 제거하는 편이 안전하다. `viewport-fit=cover`가 없던 앱에는 이 변경이 그대로 발생하지 않는다.

## Flutter 3.50에서 달라진 조건

Flutter Web은 이전에 `viewport-fit=cover`를 무시하고 `ViewPadding.zero`를 보고하는 경우가 많았다. 3.50부터는 `index.html`의 설정을 유지하고 브라우저의 CSS `env(safe-area-inset-*)` 값을 `FlutterView.viewPadding`, `MediaQuery.paddingOf`, `SafeArea`에 전달한다. 따라서 예전에는 효과가 없던 `SafeArea`가 업그레이드 뒤 실제 여백을 적용한다.

| 프로젝트 조건 | 3.50 이후 판단 | 선택 |
| --- | --- | --- |
| `viewport-fit=cover` 없음 | 기본 안전 영역 안에서 렌더링 | 변경하지 않음 |
| `viewport-fit=cover` 있음, 버튼이 잘림 | Edge-to-edge와 인셋이 충돌 | 필요한 영역에 `SafeArea` |
| 전체 화면보다 단순한 업무 UI가 우선 | 노치 뒤 렌더링이 불필요 | `viewport-fit=contain`으로 복귀 |

그림에서 확인할 부분은 화면 전체를 채우는 것과 터치 가능한 컨트롤을 안전 영역 안에 두는 일이 별개라는 점이다.

## 기존 설정부터 찾는다

아래 명령으로 문제가 될 수 있는 메타 태그를 확인한다. `viewport-fit=cover`가 없다면 이 변경만으로 레이아웃이 달라졌다고 단정하면 안 된다.

```bash
rg -n "viewport-fit|meta name=\"viewport\"" web/index.html
```

다음 형태가 기존 파일에 있다면 영향을 받을 수 있다.

```html
<meta name="viewport"
      content="width=device-width, initial-scale=1.0, viewport-fit=cover">
```

## 전체 배경은 유지하고 조작부만 보호하기

지도, 사진, 대시보드처럼 배경이 화면 끝까지 보여야 하는 앱은 `Scaffold` 전체를 무조건 감싸기보다 사용자가 누르는 영역을 보호하는 편이 낫다. 아래 코드는 콘텐츠 배경은 유지하면서 상단·하단 조작부만 안전 영역 안에 둔다는 예시다.

```dart
Scaffold(
  body: Stack(
    children: [
      const FullBleedDashboard(),
      SafeArea(
        child: Column(
          children: [
            const HeaderActions(),
            const Spacer(),
            const BottomActions(),
          ],
        ),
      ),
    ],
  ),
);
```

화면 전체가 잘려도 안 되는 일반 폼이라면 `Scaffold`의 `body`를 `SafeArea`로 감싸는 구성이 단순하다. `SafeArea` 안에서 다시 `MediaQuery.paddingOf(context)`를 더하면 같은 인셋을 중복 적용할 수 있으니 둘 중 하나를 기준으로 삼는다.

## cover가 필요 없다면 설정을 되돌린다

PWA가 아니고 모바일 브라우저에서 굳이 노치 영역까지 사용할 이유가 없다면 메타 태그를 기본값으로 바꾼다.

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

이 선택은 `SafeArea` 코드를 전부 삭제하라는 뜻이 아니다. 모바일 앱과 웹이 같은 위젯 트리를 쓰거나 특정 화면만 edge-to-edge라면, 웹에서도 필요한 컨트롤 주변의 `SafeArea`를 유지하는 편이 플랫폼 차이를 줄인다.

## 출시 전 확인 목록

- [ ] `web/index.html`에 `viewport-fit=cover`가 실제로 있는지 확인했다.
- [ ] Flutter 3.50 release 빌드를 iOS Safari와 Android Chrome에서 열었다.
- [ ] 노치가 있는 기기에서 상단 버튼과 하단 CTA를 눌러봤다.
- [ ] `SafeArea`를 적용한 뒤 배경이 불필요하게 잘리지 않는지 확인했다.
- [ ] cover를 제거했다면 PWA 설치 화면의 전체 배경 요구사항을 다시 확인했다.

확인 기준은 2026년 10월 31일 Flutter 공식 문서다. 이 문제는 Dart 코드 오류나 브라우저 캐시 문제가 아니라 Flutter Web 엔진이 `viewport-fit=cover`와 브라우저 안전 영역을 해석하는 방식의 변화다. 기존 설정이 없는 앱에 임의로 `SafeArea`를 전역 적용하면 오히려 여백이 늘 수 있다.

참고 문서: [Flutter Web viewport-fit=cover 변경 안내](https://docs.flutter.dev/release/breaking-changes/viewport-fit-cover-safe-area), [Flutter Web 배포 문서](https://docs.flutter.dev/deployment/web), [Flutter SafeArea·MediaQuery](https://docs.flutter.dev/ui/adaptive-responsive/safearea-mediaquery)
