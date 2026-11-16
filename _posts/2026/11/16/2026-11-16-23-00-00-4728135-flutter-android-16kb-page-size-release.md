---
layout: post
title: "Flutter Android 16 KB page size 오류 - 네이티브 플러그인 출시 점검"
description: "Flutter 앱의 Google Play 16 KB page size 경고를 네이티브 플러그인 문제로 좁히고, NDK r28 업그레이드와 AAB 검증 순서로 출시 조건을 확인한다."
date: 2026-11-16
tags: [Flutter, Android, 배포·운영, 성능최적화, CI/CD]
comments: true
share: true
---
![Flutter Android 16 KB page size 출시 점검](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

그림에서 볼 부분은 Flutter 코드가 아니라 Google Play에 올라가는 AAB 안의 네이티브 라이브러리다.

Google Play에서 `Your app uses native libraries that don't support 16 KB memory page sizes` 경고를 받은 Flutter 앱이라면, Dart 코드를 고치기보다 `*.so`를 포함한 플러그인과 Flutter SDK를 확인하는 편이 빠르다. 네이티브 코드를 전혀 포함하지 않는 순수 Dart 앱에는 이 글의 NDK 수정이 필요하지 않다. 반대로 카메라·지도·DB·ML 플러그인이 들어간 앱은 빌드가 성공해도 AAB 검사에서 막힐 수 있다.

## 왜 Flutter 앱에서 발생하나

Android 15부터 16 KB 메모리 페이지 기기를 지원할 수 있고, Google Play는 Android 15 이상을 대상으로 하는 네이티브 코드 포함 앱에 호환성을 요구한다. Android 공식 문서 기준으로 2027년 2월 1일부터는 호환되지 않는 업데이트를 출시할 수 없다. 2026년 11월 현재 새 업데이트를 준비한다면 경고를 무시하고 미루기보다 이번 빌드에서 확인하는 편이 안전하다.

Flutter 엔진 자체만의 문제라고 단정하면 안 된다. 오래된 AAR, C/C++ 플러그인, 사전 빌드된 `arm64-v8a/*.so`가 4 KB 정렬로 남아 있는 경우가 더 까다롭다.

| 확인 대상 | 판단 | 대응 |
|---|---|---|
| `*.so`가 없음 | Java/Kotlin 또는 순수 Dart 중심 | NDK 변경 없이 AAB만 점검 |
| Flutter SDK가 오래됨 | 엔진·기본 NDK가 오래될 수 있음 | Flutter 3.38 이상으로 업그레이드 검토 |
| 플러그인 `.so`만 실패 | 의존성의 사전 빌드 바이너리 문제 | 해당 플러그인 업데이트 또는 대체 |
| 직접 만든 JNI/NDK 코드 실패 | 링크 정렬과 도구체인 문제 | NDK r28 이상으로 재빌드 |

## 우선 Flutter와 NDK 조건을 맞춘다

Flutter 3.38은 Android 네이티브 코드의 16 KB 정렬을 위한 기본 NDK를 r28로 올렸다. 따라서 오래된 프로젝트에서 `android/app/build.gradle`의 NDK를 무작정 바꾸기보다 Flutter 버전, AGP, Gradle 조합을 함께 확인해야 한다.

프로젝트에서 실제로 사용하는 값부터 확인하는 명령은 다음과 같다.

```bash
flutter --version
rg "ndkVersion|compileSdk|targetSdk" android
flutter pub outdated
```

직접 NDK를 빌드하는 플러그인이라면 `android/app/build.gradle` 또는 플러그인 모듈의 설정을 확인한다. NDK 버전만 올려도 소스가 재컴파일되는 프로젝트와, 저장소에 미리 빌드된 `.so`를 계속 복사하는 프로젝트는 결과가 다르다. 후자는 패키지 제공자가 새 바이너리를 배포해야 해결된다.

## AAB 안에서 실패한 라이브러리를 찾는다

릴리스 빌드가 끝난 뒤에는 소스가 아니라 최종 산출물을 확인해야 한다. 아래처럼 AAB를 임시 디렉터리에 풀면 어떤 ABI의 라이브러리가 들어갔는지 추적할 수 있다.

```bash
flutter clean
flutter pub get
flutter build appbundle --release

rm -rf /tmp/flutter-aab-check
mkdir -p /tmp/flutter-aab-check
unzip -q build/app/outputs/bundle/release/app-release.aab -d /tmp/flutter-aab-check
find /tmp/flutter-aab-check -name '*.so' -print
```

APK를 만들 수 있는 환경이라면 `zipalign`으로 패키지 정렬도 확인한다.

```bash
flutter build apk --release
zipalign -c -P 16 -v 4 build/app/outputs/flutter-apk/app-release.apk
```

이 명령이 성공해도 모든 네이티브 코드가 16 KB에서 정상 실행된다는 뜻은 아니다. AAB의 각 ABI를 Android Studio APK Analyzer 또는 16 KB 시스템 이미지에서 확인해야 한다. 실패한 파일명이 특정 플러그인 경로를 가리키면, 앱 코드에 임시 플래그를 추가하기보다 그 플러그인의 릴리스 노트와 이슈에서 16 KB 지원 버전을 찾아야 한다.

## 출시 전 체크리스트

- [ ] `flutter --version`이 프로젝트가 의도한 안정 버전인가
- [ ] 앱과 플러그인에 사전 빌드된 `arm64-v8a/*.so`가 있는가
- [ ] NDK를 올린 뒤 실제로 네이티브 모듈이 재컴파일됐는가
- [ ] 최종 AAB에서 ABI별 `.so` 목록을 확인했는가
- [ ] APK Analyzer 또는 16 KB 에뮬레이터에서 설치·실행했는가
- [ ] Play Console 경고가 사라진 동일 버전의 AAB를 업로드했는가

무료로 해결되는 조건도 있다. 네이티브 플러그인이 사용되지 않는 앱이라면 NDK를 추가 설치하거나 패키지를 구매할 이유가 없다. 반대로 오래된 상용 SDK가 문제라면 설정 한 줄보다 SDK 교체·업데이트 비용이 더 크므로, 출시 직전에 전체 플러그인을 한꺼번에 올리지 말고 실패한 `.so`를 기준으로 하나씩 바꾸는 편이 복구하기 쉽다.

핵심은 `flutter build` 성공 여부가 아니라 Google Play에 제출할 AAB 내부의 네이티브 바이너리다. Flutter 3.38과 NDK r28은 출발점일 뿐이며, 최종 판단은 플러그인 바이너리와 16 KB 환경 실행 결과로 내려야 한다.

참고 기준은 [Android 16 KB page size 공식 문서](https://developer.android.com/guide/practices/page-sizes), [Google Play 기술 품질 요구사항](https://support.google.com/googleplay/android-developer/answer/17492799), [Flutter 3.38 변경 사항](https://flutter.dev/blog/whats-new-in-flutter-3-38)이다.
