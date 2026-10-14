---
layout: post
title: "Flutter Android 13 테마 아이콘 오류 - release에서 단색 아이콘이 안 보일 때"
description: "Flutter 앱의 Android 13 테마 아이콘이 기본 아이콘으로 보일 때 adaptive icon의 monochrome 레이어와 release 리소스를 점검하는 방법을 정리한다."
date: 2026-10-15
tags: [Flutter, Android, 출시, 접근성]
comments: true
share: true
---

![Flutter Android 13 adaptive icon과 monochrome 레이어](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

Android 13 이상에서 테마 아이콘을 지원하려는 Flutter 앱이라면 `android:icon`만 바꾸는 방식보다 adaptive icon에 `monochrome` 레이어를 추가하는 선택이 맞다. 테마 아이콘을 쓰지 않는 사용자나 Android 12 이하까지 같은 모양을 유지해야 하는 앱에는 필수 변경이 아니다. 이 문제는 빌드 실패보다 출시 후 런처에서 아이콘이 단색으로 바뀌지 않거나, foreground 이미지가 잘리는 형태로 드러난다.

## 왜 release에서만 다르게 보이나

Android 13(API 33)부터 사용자가 테마 아이콘을 켜고 런처가 지원하면 adaptive icon의 `monochrome` 레이어가 배경색에 맞춰 색칠된다. Flutter 프로젝트의 기본 아이콘은 보통 `@mipmap/ic_launcher`를 가리키지만, 기존 PNG 하나를 교체하는 것만으로는 이 레이어가 생기지 않는다.

특히 debug에서 확인한 아이콘과 Play 설치본이 다르면 다음 값을 분리해서 봐야 한다.

| 확인 대상 | 흔한 결과 | 원인 |
|---|---|---|
| Android 12 이하 | 예전 아이콘 | legacy PNG 또는 API 26 미만 리소스가 선택됨 |
| Android 13 테마 아이콘 끔 | 컬러 adaptive 아이콘 | foreground/background는 정상 |
| Android 13 테마 아이콘 켬 | 기본 아이콘·단색 누락 | `monochrome` 레이어 없음 |
| release 설치본 | debug와 다른 아이콘 | flavor 리소스 또는 `application`의 icon 경로 불일치 |

## `flutter_launcher_icons`를 쓰는 설정

이미 이 패키지로 아이콘을 생성하는 프로젝트라면 설정에 세 가지 입력을 분리한다. 아래 코드는 컬러 adaptive icon과 Android 13 테마용 단색 전경을 함께 만들기 위해 필요하다.

```yaml
flutter_launcher_icons:
  android: true
  ios: true
  image_path: assets/icon/app_icon.png
  adaptive_icon_background: "#FFFFFF"
  adaptive_icon_foreground: assets/icon/app_icon_foreground.png
  adaptive_icon_monochrome: assets/icon/app_icon_monochrome.png
```

`adaptive_icon_background`와 `adaptive_icon_foreground` 중 하나만 넣으면 adaptive icon이 완성되지 않는다. `adaptive_icon_monochrome`에는 배경이 투명하고 아이콘 형태가 흰색인 이미지를 준비하는 편이 안전하다. 원본 컬러 로고를 그대로 넣으면 테마 색상으로 칠해질 때 외곽 배경까지 아이콘처럼 보일 수 있다.

설정 파일만 고친 뒤 앱을 실행하지 말고 생성 명령과 release 빌드를 분리해 확인한다.

```bash
dart run flutter_launcher_icons
flutter clean
flutter build apk --release
```

직접 리소스를 관리한다면 `android/app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml`에 레이어를 선언한다. 이 파일은 API 26 이상에서 선택되므로 Android 12 이하용 PNG를 지우면 안 된다.

```xml
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
    <monochrome android:drawable="@drawable/ic_launcher_monochrome" />
</adaptive-icon>
```

여기서 `monochrome`는 알림 아이콘 설정과 다른 리소스다. 알림 표시줄의 작은 아이콘이 흰색으로 보여야 한다는 이유로 런처 테마 아이콘 파일을 재사용하면, 홈 화면 마스크와 안전 영역이 맞지 않을 수 있다.

## 출시 전에 확인할 조건

Play에 올리는 release APK 또는 AAB의 `application`이 실제 생성한 `@mipmap/ic_launcher`를 가리키는지 확인한다. flavor를 사용한다면 `src/release`, `src/<flavor>`에 같은 이름의 리소스가 덮어쓰지 않는지도 본다. 아이콘 생성 패키지를 추가했지만 AndroidManifest의 icon 이름을 별도로 바꿨다면 생성 결과가 사용되지 않는다.

체크 순서는 아래처럼 짧게 고정하면 된다.

- [ ] 컬러 adaptive icon에 background와 foreground가 모두 있다.
- [ ] Android 13 테마 아이콘용 `monochrome` 레이어가 있다.
- [ ] foreground와 monochrome 이미지의 투명 여백과 중앙 안전 영역을 확인했다.
- [ ] Android 12, Android 13 테마 아이콘 켬, 테마 아이콘 끔에서 release 설치본을 확인했다.
- [ ] flavor별 Manifest와 `mipmap-anydpi-v26` 리소스가 다른 파일을 가리키지 않는다.
- [ ] 아이콘을 바꾼 뒤 기존 런처 캐시를 피하려고 앱 삭제·재설치로 한 번 더 확인했다.

Android 공식 문서 기준으로 adaptive icon은 foreground·background 두 레이어가 필요하고, 테마 아이콘을 지원하려면 monochrome 단일 레이어를 추가한다. Flutter 앱의 Android 아이콘도 네이티브 Android 리소스 방식으로 갱신되므로, Dart 코드가 정상이어도 `res/mipmap`과 Manifest가 틀리면 화면에서 결과가 달라진다. 확인 기준일은 2026년 10월 15일이다.

참고 문서: [Android adaptive icons 공식 문서](https://developer.android.com/develop/ui/compose/system/icon_design_adaptive), [Flutter 앱 아이콘과 이미지 리소스](https://docs.flutter.dev/ui/assets/assets-and-images), [`flutter_launcher_icons` 설정](https://pub.dev/packages/flutter_launcher_icons)
