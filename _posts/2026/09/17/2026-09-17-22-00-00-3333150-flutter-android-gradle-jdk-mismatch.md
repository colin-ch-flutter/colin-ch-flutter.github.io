---
layout: post
title: "Flutter Android Gradle JDK 오류 - Android Studio 업데이트 뒤 출시 빌드 복구 순서"
description: "Flutter Android 릴리스 빌드에서 Java·Gradle·AGP가 맞지 않을 때 flutter doctor와 analyze로 원인을 좁히고 안전하게 복구하는 방법을 정리한다."
date: 2026-09-17
tags: [Flutter, Android, CI/CD, 성능최적화]
comments: true
share: true
---

![Flutter Android 릴리스 빌드의 Java와 Gradle 버전 불일치를 점검하는 터미널 화면](https://images.unsplash.com/photo-1515879218367-8466d910aaa4?w=1200&q=80)

그냥 Android Studio를 최신 버전으로 올린 뒤 `flutter build appbundle`이 깨졌다면 JDK를 무작정 바꾸기보다 Flutter가 실제로 선택한 Java와 프로젝트의 Gradle wrapper를 함께 확인해야 한다. 오래된 Android 프로젝트는 Gradle만 올리는 편이 빠를 수 있지만, 이미 최신 AGP를 쓰는 프로젝트는 JDK 경로를 고정하는 편이 변경 범위가 작다. iOS 빌드만 필요한 개발자나 새 Flutter 프로젝트가 정상 빌드되는 경우에는 이 글의 수정이 필요 없다.

## 오류의 핵심은 세 버전의 연결이다

Flutter Android 빌드는 Dart 코드만으로 끝나지 않는다. Flutter가 선택한 JDK가 Gradle을 실행하고, Gradle이 Android Gradle Plugin(AGP)을 로드한다. 셋 중 하나만 바뀌면 `Your project's Gradle version is incompatible with the Java version that Flutter is using` 또는 `Unsupported class file major version` 같은 오류가 난다.

| 확인 대상 | 확인 명령 또는 파일 | 판단 기준 |
|---|---|---|
| Flutter가 쓰는 Java | `flutter doctor -v` | 출력된 Java 경로와 버전이 의도한 JDK인지 |
| 조합 제안 | `flutter analyze --suggestions` | AGP·Java·Gradle 호환성 경고가 있는지 |
| Gradle wrapper | `android/gradle/wrapper/gradle-wrapper.properties` | `distributionUrl`의 Gradle 버전 |
| AGP | `android/settings.gradle` 또는 `build.gradle` | `com.android.application` 버전 |

Flutter 공식 문서도 Android Studio에 포함된 JDK를 기본으로 사용한다고 설명한다. 따라서 셸의 `java -version`만 확인하면 부족하다. Android Studio를 업데이트한 뒤에는 `flutter doctor -v`의 `Java binary at` 경로를 기준으로 봐야 한다.

## 복구 순서

오류가 난 프로젝트 루트에서 아래 명령으로 실제 환경과 권장 조합을 기록한다. 이 출력은 CI 환경과 비교할 때도 유용하다.

```bash
flutter doctor -v
flutter analyze --suggestions
cd android
./gradlew --version
cd ..
```

`flutter analyze --suggestions`가 특정 Gradle 버전을 제안하면 그 값을 우선한다. 문서에 나온 과거의 “Gradle 7.3~7.6.1” 범위를 모든 프로젝트에 복사하면 안 된다. 이 범위는 Java 17로 인해 깨진 구형 프로젝트를 위한 마이그레이션 안내이며, 현재 AGP가 요구하는 Gradle과 함께 판단해야 한다.

JDK를 바꿔야 한다면 Flutter 전체에 적용할 경로를 명시한다. macOS 예시는 Android Studio의 JBR 경로를 실제 설치 위치로 바꿔야 한다.

```bash
flutter config --jdk-dir="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
flutter doctor -v
flutter clean
flutter pub get
flutter build appbundle --release
```

반대로 프로젝트가 오래된 Gradle을 고정하고 있고 JDK 17에서만 실패한다면 Android 폴더를 Android Studio로 열어 AGP Upgrade Assistant를 사용하거나, 제안된 wrapper 버전으로만 올린다. `android/gradle-wrapper.properties`와 AGP 버전을 한 번에 임의로 최신화하면 Kotlin 플러그인이나 서드파티 플러그인 오류로 문제가 옮겨갈 수 있다.

## CI에서 다시 깨지지 않게 하는 체크리스트

- 로컬과 CI의 `flutter --version`, Java 경로, `./gradlew --version`을 저장한다.
- Android Studio의 JDK와 `JAVA_HOME`이 서로 다르면 어느 쪽을 표준으로 삼을지 정한다.
- `flutter analyze --suggestions` 결과를 릴리스 브랜치 병합 전에 확인한다.
- 수정 후 `flutter build appbundle --release`까지 실행하고 debug 빌드만 성공한 상태로 배포하지 않는다.
- wrapper와 Gradle 캐시를 지우기 전에 현재 버전 파일을 커밋해 되돌릴 수 있게 한다.

여기서 헷갈리기 쉬운 점은 `sourceCompatibility`를 Java 17로 설정하는 것과 Gradle을 실행하는 JDK를 Java 17로 고정하는 것이 같은 작업이 아니라는 사실이다. 전자는 앱 소스의 컴파일 대상이고, 후자는 빌드 도구 자체의 실행 환경이다. 오류 로그에 `Unsupported class file`이나 `Gradle version is incompatible`가 있으면 후자부터 확인한다.

요점은 세 가지다. `java -version` 대신 `flutter doctor -v`로 Flutter의 JDK를 확인하고, `flutter analyze --suggestions`로 조합을 검증하며, 수정 범위를 JDK 경로 또는 wrapper 중 하나로 제한한다. 이 순서를 지키면 Android Studio 업데이트를 되돌리지 않고도 출시 빌드 복구 여부를 빠르게 판단할 수 있다.

참고: [Flutter Android Java·Gradle 마이그레이션 가이드](https://docs.flutter.dev/release/breaking-changes/android-java-gradle-migration-guide), [Flutter Android 빌드 및 출시](https://docs.flutter.dev/deployment/android), [Android 빌드의 Java 버전](https://developer.android.com/build/jdks)
