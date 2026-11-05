---
layout: post
title: "Flutter Android 17 UnsatisfiedLinkError 오류 - ReLinker native library 로딩 조건"
description: "Flutter 앱을 targetSdk 37로 올린 뒤 Android 17에서 UnsatisfiedLinkError가 발생한다면 System.load, ReLinker, Flutter SDK와 native 플러그인 업데이트 조건을 분리해 점검하는 방법을 정리한다."
date: 2026-11-06
tags: [Flutter, Android, 플랫폼통합, 배포운영, 성능최적화]
comments: true
share: true
---

![Flutter Android 17 native library 로딩 오류 점검](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

Android 17에서만 Flutter 앱이 시작 직후 `UnsatisfiedLinkError`로 종료된다면, `targetSdk 37`과 native 라이브러리 로더를 우선 확인해야 한다. Android 17 기기에서 실행했다는 사실만으로 모든 앱이 영향을 받는 것은 아니다. `System.load()`로 `.so`를 읽거나 오래된 Flutter 엔진·플러그인이 ReLinker를 사용하는 출시 앱이 대상이다. 일반 Dart 앱이거나 target SDK가 36 이하라면 이 오류를 해결하려고 권한 코드를 추가할 필요가 없다.

## 오류가 생기는 조건

Android 공식 문서 기준으로 target API 37 이상 앱은 native 파일을 `System.load()`로 읽을 때 파일이 read-only가 아니면 `UnsatisfiedLinkError`가 발생한다. Flutter 쪽 이슈에서도 ReLinker가 fallback 과정에서 파일 권한을 프로그램matically 바꾸는 동작이 Android 17의 조건과 충돌한다고 설명한다.

| 확인 항목 | 영향 | 판단 |
|---|---|---|
| 기기 OS | Android 17 | 재현 환경이지만 단독 원인은 아니다 |
| `targetSdk` | 37 이상 | 새 native DCL 규칙이 적용된다 |
| 로더 | `System.load()` 또는 ReLinker | 파일 권한 변경 여부를 확인한다 |
| 코드 | Flutter 엔진·FFI·native 플러그인 | 순수 Dart 화면만 쓰면 가능성이 낮다 |

로그에는 보통 `java.lang.UnsatisfiedLinkError`와 함께 `libflutter.so`, 플러그인의 `lib*.so`, 또는 로더 호출 스택이 보인다. `libflutter.so`가 보이면 앱 코드의 Dart 예외가 아니라 Flutter 엔진을 로드하는 단계에서 실패한 것이다.

## Flutter SDK와 플러그인부터 업데이트한다

Flutter 저장소의 Android 17 대응 이슈는 ReLinker로 `libflutter.so`를 로드하지 않도록 수정하는 작업으로 닫혀 있다. 따라서 `build.gradle`에 임의의 권한 설정을 붙이는 것보다 stable 채널의 Flutter SDK와 native 플러그인을 올리고 같은 소스에서 다시 빌드하는 순서가 맞다.

버전을 기록하고 캐시가 섞이지 않은 release 산출물을 만드는 명령은 다음과 같다.

```bash
flutter channel stable
flutter upgrade
flutter --version
flutter pub outdated
flutter pub upgrade
flutter clean
flutter pub get
flutter build appbundle --release
```

실제 프로젝트에서 `flutter upgrade`를 바로 실행하기 어렵다면 현재 버전, `pubspec.lock`, 오류가 난 native 플러그인 버전을 먼저 저장한다. 플러그인 changelog에 Android 17, ReLinker, `System.loadLibrary` 관련 수정이 있는지도 확인해야 한다. 플러그인만 업데이트하고 Flutter 엔진은 그대로 두면 `libflutter.so` 로딩 오류가 남을 수 있다.

## 내 native 코드라면 read-only를 깨지 않는다

직접 만든 Android 플러그인이나 FFI 로더에서 아래처럼 임시 파일로 복사한 뒤 권한을 바꾸는 방식은 Android 17에서 문제가 될 수 있다.

```kotlin
// native .so를 앱 디렉터리에 복사한 뒤 chmod로 실행 조건을 바꾸는 로더는 피한다.
// 가능한 경우 APK의 표준 native library 경로에서 로드한다.
System.loadLibrary("my_native_library")
```

이 코드는 모든 프로젝트에 그대로 넣는 만능 수정안이 아니다. 라이브러리가 APK의 `lib/<ABI>/`에 정상 패키징되어 있고 이름이 맞을 때만 `System.loadLibrary` 경로를 검토한다. 파일을 앱 저장소로 복사하는 기존 로더라면 해당 SDK의 Android 17 대응 버전을 우선 찾고, 직접 유지하는 코드라면 복사 후 `chmod`나 쓰기 작업을 제거하는 쪽으로 구조를 바꾼다.

APK 안에 라이브러리가 실제로 들어갔는지도 따로 확인한다. 이 검사는 DCL 문제와 ABI 누락 문제를 구분하는 데 도움이 된다.

```bash
apkanalyzer files list app-release.apk | rg 'libflutter.so|libapp.so|lib.*\.so'
unzip -l app-release.apk | rg 'lib/(arm64-v8a|armeabi-v7a|x86_64)/.*\.so'
adb logcat -c
adb logcat | rg -i 'UnsatisfiedLinkError|System\.load|ReLinker|libflutter|FATAL EXCEPTION'
```

`libflutter.so`가 특정 ABI에만 없으면 `abiFilters`나 flavor별 native 파일 패키징 문제일 수 있다. 16KB page size 오류, 32비트 라이브러리 오류, read-only DCL 오류는 로그의 문구와 발생 조건이 다르므로 하나의 해결책으로 묶지 않는다.

## 출시 전 판단 체크리스트

- [ ] `targetSdk`가 37 이상인지 기록했다.
- [ ] Android 17 실기기에서 release APK 또는 AAB 설치본을 확인했다.
- [ ] `flutter --version`과 native 플러그인 버전을 빌드 로그에 남겼다.
- [ ] `UnsatisfiedLinkError`의 대상이 Flutter 엔진인지 특정 플러그인인지 확인했다.
- [ ] ReLinker가 오래된 버전이면 업데이트하거나 유지보수자 수정본을 적용했다.
- [ ] APK의 출시 ABI마다 `libflutter.so`, `libapp.so`, 플러그인 `.so`가 존재한다.
- [ ] 파일 권한을 바꾸는 post-build 스크립트를 제거하거나 별도 검토했다.

정리하면 이 오류는 Android 17 권한을 Flutter에서 요청하는 문제가 아니다. `targetSdk 37`의 native DCL 규칙, 파일을 로드하는 주체, Flutter 엔진·플러그인 버전을 나눠야 한다. Flutter SDK와 플러그인을 최신 stable로 올린 뒤 release 산출물의 ABI와 logcat을 함께 확인하고, 직접 만든 로더에서 `.so`를 복사·변경하는 동작이 있다면 그 부분을 없애는 것이 출시 복구의 기준이다.

참고 문서: [Android 17 동작 변경](https://developer.android.com/about/versions/17/behavior-changes-17), [Flutter Android 17 DCL 이슈](https://github.com/flutter/flutter/issues/184861), [Flutter Android 빌드·배포 문서](https://docs.flutter.dev/deployment/android)
