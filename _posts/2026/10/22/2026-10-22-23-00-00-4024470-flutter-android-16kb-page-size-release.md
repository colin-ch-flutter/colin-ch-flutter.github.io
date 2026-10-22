---
layout: post
title: "Flutter Android 16 KB 페이지 오류 - release AAB를 막는 네이티브 플러그인 찾기"
description: "Flutter Android 앱이 Google Play의 16 KB 페이지 호환성 검사에서 막힐 때 순수 Dart 앱과 네이티브 플러그인을 구분하고, 문제의 .so 파일을 찾아 업데이트·재빌드·검증하는 출시 절차를 정리한다."
date: 2026-10-22
tags: [Flutter, Android, CI/CD, 성능최적화]
comments: true
share: true
---

![Flutter Android release AAB의 16 KB 페이지 호환성을 점검하는 개발 화면](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

Android 15 이상을 대상으로 Flutter 앱을 출시하는데 Play Console에서 `16 KB page size` 경고가 나온다면, 순수 Dart 코드보다 번들 안의 네이티브 `.so` 파일을 먼저 의심해야 한다. 네이티브 코드를 전혀 포함하지 않는 앱은 대체로 추가 수정이 필요 없지만, 카메라·PDF·SQLite·암호화·머신러닝 플러그인을 사용한다면 특정 플러그인 하나가 AAB 전체의 호환성 검사를 막을 수 있다.

## 왜 debug는 되고 release에서 막히나

Android 15부터 16 KB 메모리 페이지 크기를 쓰는 기기가 지원된다. Android 공식 문서는 NDK 라이브러리를 직접 또는 SDK를 통해 포함한 앱은 다시 빌드해야 할 수 있다고 설명한다. Google Play의 최신 안내 기준으로 Android 15(API 35) 이상을 타깃으로 하는 앱 업데이트는 16 KB 기기 호환성을 요구하며, 2027년 2월 1일부터 미지원 업데이트는 출시할 수 없다.

Flutter 엔진 자체가 정상이어도 `flutter_pdfview`, 이미지 처리, 지도, 로컬 DB 같은 플러그인이 오래된 ELF 바이너리를 포함하면 문제가 남는다. 그래서 debug APK가 실행된다는 사실만으로 release AAB가 통과한다고 판단하면 안 된다. debug와 release가 서로 다른 ABI, 압축 방식, 의존성 버전을 사용할 수 있기 때문이다.

| 확인 대상 | 판단 | 대응 |
|---|---|---|
| `libflutter.so`와 앱에 `.so`가 없음 | 순수 Dart 또는 Java/Kotlin 중심 | 16 KB 환경에서 회귀 테스트 |
| 특정 플러그인의 `.so`만 정렬 불일치 | 플러그인이 병목 | 해당 플러그인 업데이트·교체·소스 재빌드 |
| 여러 `.so`가 오래된 Flutter/NDK에서 생성됨 | 빌드 도구 조합 문제 | Flutter·NDK·AGP를 호환 세트로 업그레이드 |

그림에서 봐야 할 부분은 APK가 실행되는지보다 최종 산출물 안에 어떤 네이티브 라이브러리가 들어갔는지다.

## AAB를 만들고 문제 라이브러리를 좁힌다

Play에 올리기 전 release 산출물의 ABI를 고정한다. 아래 명령은 예시이며 실제 flavor를 사용한다면 `--flavor production`을 추가한다.

```bash
flutter clean
flutter pub get
flutter build appbundle --release --target-platform android-arm64
```

생성된 AAB는 Android Studio의 APK Analyzer에서 열거나, bundletool로 기기용 APK를 만든 뒤 검사한다. APK Analyzer에서 `lib/arm64-v8a/`를 펼쳐 `.so` 파일의 출처를 확인하면 플러그인 후보를 빠르게 줄일 수 있다. 파일명이 모호하면 `pubspec.lock`의 플러그인과 `android/.gradle` 의존성을 함께 비교한다.

APK를 직접 검사할 수 있는 환경이라면 ELF와 ZIP 정렬도 확인한다.

```bash
unzip -q build/app/outputs/flutter-apk/app-release.apk -d /tmp/flutter-release-apk
find /tmp/flutter-release-apk/lib -name '*.so' -print

# Android SDK의 build-tools 경로에 있는 zipalign 사용
zipalign -c -P 16 -v 4 build/app/outputs/flutter-apk/app-release.apk

# NDK readelf로 각 ELF LOAD segment 정렬 확인
"$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/darwin-x86_64/bin/llvm-readelf" \\
  -l /tmp/flutter-release-apk/lib/arm64-v8a/문제파일.so | rg 'LOAD|Align'
```

여기서 macOS가 아닌 CI라면 `darwin-x86_64` 부분을 실행 중인 호스트의 NDK 경로로 바꾼다. `zipalign` 통과만으로 모든 런타임 문제가 사라지는 것은 아니다. Android 공식 문서가 안내하는 16 KB 에뮬레이터에서 설치·실행까지 확인해야 한다.

## 복구 순서와 멈춰야 할 지점

문제 파일이 플러그인에서 왔다면 버전만 올리지 말고 릴리스 노트에서 16 KB 지원 여부와 최소 Android SDK 조건을 확인한다. 업데이트 후에도 같은 `.so`가 남으면 `pubspec.lock`이 실제로 바뀌었는지, 캐시된 Gradle 산출물이 재사용되지 않았는지 확인한다.

1. 문제가 된 패키지의 최신 호환 버전을 확인한다.
2. `flutter pub upgrade <패키지명>` 후 `flutter clean`으로 release를 다시 만든다.
3. 여전히 같은 파일이면 해당 플러그인의 GitHub 이슈와 transitive 의존성을 확인한다.
4. 대체 패키지를 선택할 때 기능만 비교하지 말고 네이티브 `.so` 제공 여부와 유지 상태를 비교한다.
5. Flutter를 올릴 경우 Dart·AGP·Gradle·Kotlin·Java·NDK를 한 번에 호환 세트로 검증한다.

Flutter 3.38 안내는 Java 17, AGP 8.11.1, Gradle 8.14, NDK r28 조합을 CI에서 확인한 예로 제시한다. 이것은 모든 프로젝트에 그대로 복사할 버전표가 아니다. 오래된 앱에서 AGP만 올리면 다른 빌드 오류와 최소 SDK 변경이 생길 수 있으므로, 문제가 한 플러그인의 `.so`라면 먼저 플러그인 업데이트나 교체를 검토하는 편이 변경 범위가 작다.

## 출시 전 체크리스트

- [ ] `flutter build appbundle --release`를 실제 출시 flavor로 만들었다.
- [ ] AAB/APK의 `lib/arm64-v8a` 아래 `.so` 목록과 출처를 확인했다.
- [ ] Play Console App bundle 탐색기에서 16 KB 호환성 결과를 확인했다.
- [ ] 16 KB Android 에뮬레이터 또는 테스트 기기에서 설치·실행·핵심 플러그인 기능을 확인했다.
- [ ] 문제가 된 플러그인의 버전과 대체·롤백 계획을 기록했다.

16 KB 경고가 없는 앱이라면 빈 설정을 추가하거나 무리하게 Flutter 버전을 올릴 필요는 없다. 반대로 네이티브 플러그인이 있는 앱은 “내 폰에서 실행된다”보다 “Play에 제출할 AAB가 검사에 통과한다”를 출시 기준으로 삼아야 한다. 확인 기준은 [Android 16 KB 페이지 크기 공식 문서](https://developer.android.com/guide/practices/page-sizes), [Flutter 3.38의 Android 빌드 안내](https://flutter.dev/blog/whats-new-in-flutter-3-38), [Google Play 호환성 공지](https://android-developers.googleblog.com/2025/05/prepare-play-apps-for-devices-with-16kb-page-size.html)다.
