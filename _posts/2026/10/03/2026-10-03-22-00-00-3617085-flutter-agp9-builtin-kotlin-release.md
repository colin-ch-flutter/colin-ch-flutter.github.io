---
layout: post
title: "Flutter AGP 9 built-in Kotlin 오류 - 플러그인 때문에 release 빌드가 깨질 때"
description: "Flutter 3.44 이후 AGP 9와 built-in Kotlin을 적용할 때 release 빌드가 깨지는 조건을 구분하고, android.builtInKotlin 플래그와 플러그인 점검 순서로 복구하는 방법을 정리한다."
date: 2026-10-03
tags: [Flutter, Android, Kotlin, CI/CD, 배포]
comments: true
share: true
---

![Flutter AGP 9 built-in Kotlin release 빌드 점검](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

Flutter 앱이 debug에서는 빌드되는데 AGP 9로 올린 뒤 release에서 `org.jetbrains.kotlin.android` 충돌이 난다면, 앱 코드보다 플러그인의 Gradle 설정을 봐야 한다. Flutter 3.44 앱은 `android.builtInKotlin=false`로 AGP 9를 임시 사용할 수 있고, 실제 built-in Kotlin 전환은 Flutter 3.47 이상과 전체 플러그인 마이그레이션이 가능한 경우에만 선택하는 편이 안전하다. Kotlin을 전혀 쓰지 않는 앱이라면 이 전환을 억지로 할 필요가 없다.

## 왜 debug보다 release에서 늦게 드러나나

AGP 9는 Kotlin 컴파일 지원을 기본 제공하므로 `kotlin-android` 플러그인을 별도로 적용하지 않는 방향으로 바뀌었다. 그런데 `camera`, `image_picker`, `google_sign_in`, `shared_preferences` 같은 Flutter 플러그인 중 하나라도 예전 KGP를 직접 적용하거나 `kotlin {}` 블록을 무조건 실행하면 Gradle 설정 단계에서 실패할 수 있다. 앱의 Dart 코드는 그대로여도 의존성 그래프가 달라지면 release 빌드가 막힌다.

| 상황 | 선택 | 피해야 할 판단 |
|---|---|---|
| Flutter 3.44, AGP 9 업그레이드 직후 | `android.builtInKotlin=false` 유지 | 플래그를 지우고 바로 전환 |
| Flutter 3.47 이상, 모든 플러그인 전환 확인 | built-in Kotlin 마이그레이션 | pub 캐시의 Gradle 파일 직접 수정 |

## 출시를 막는 플러그인 찾기

오류 메시지의 첫 줄만 보고 앱의 `android/app`만 고치면 원인을 놓친다. `pubspec.lock`의 실제 플러그인 버전과 Android Gradle 파일의 KGP 적용 여부를 확인한다.

앱 루트에서 아래 명령으로 의존성과 경고를 분리해 확인한다.

```bash
flutter pub deps --style=compact
flutter build appbundle --release
rg -n 'kotlin-android|org\.jetbrains\.kotlin\.android|kotlin\s*\{' \
  "$HOME/.pub-cache/hosted/pub.dev" -g 'build.gradle' -g 'build.gradle.kts'
```

검색 결과는 많을 수 있으므로 Gradle 오류에 나온 플러그인 디렉터리부터 좁혀 본다. 지원 버전이 있으면 올리고, 없다면 대체 플러그인이나 포크를 검토한다. pub 캐시를 직접 고치면 CI의 다음 `pub get`에서 사라진다.

## 당장 release를 복구하는 설정

플러그인 전체를 아직 바꾸지 못한 상태라면 Flutter가 안내하는 호환 경로를 명시한다. 이 설정은 영구 해결이 아니라, 플러그인을 순차적으로 올리는 동안 빌드를 살리는 선택이다.

```properties
# android/gradle.properties
android.builtInKotlin=false
android.newDsl=false
```

Flutter 3.44는 AGP 9에서 레거시 KGP를 잠시 사용할 수 있도록 이 경로를 지원한다. `flutter run` 또는 `flutter build apk` 뒤에 플래그가 자동으로 추가될 수 있으니 CI에서도 확인한다. AGP 변경과 플러그인 업그레이드는 커밋을 나눠야 원인 추적이 쉽다.

## built-in Kotlin으로 옮길 때의 조건

앱이 KGP를 직접 사용하지 않는다면 이 마이그레이션을 시작하지 않아도 된다. 사용하는 경우에는 AGP 9 이상, Flutter 3.47 이상, 전체 플러그인 전환을 맞춘 뒤 앱 모듈의 `kotlin-android`를 제거하고 아래 플래그를 적용한다.

```properties
# 전체 플러그인 전환을 확인한 뒤에만 적용
android.builtInKotlin=true
android.newDsl=true
```

`kotlinOptions {}`나 오래된 `applicationVariants` 접근도 함께 실패할 수 있다. 마이그레이션 뒤에는 AAB 빌드와 로그인·푸시·카메라 같은 네이티브 기능을 실제 기기에서 확인한다.

## 출시 체크리스트

- [ ] 오류에 나온 플러그인의 KGP 적용과 호환 버전을 확인했다.
- [ ] 임시 복구 플래그를 CI에도 반영했다.
- [ ] 전환 후 release AAB와 실제 네이티브 기능을 검증했다.
- [ ] 플러그인 포크의 의존성 경로를 고정했다.

AGP 9 오류를 앱의 Kotlin 버전 숫자만 바꾸는 문제로 보면 해결이 길어진다. 현재 Flutter 프로젝트에 필요한 선택은 “지금 전환 가능한가”와 “플러그인 하나가 아직 레거시 KGP에 묶였는가”를 나누는 것이다. 전환 조건이 부족하면 호환 플래그로 release를 복구하고, 플러그인별 업데이트를 끝낸 뒤 built-in Kotlin을 켜는 순서가 운영 비용이 낮다.

참고한 공식 문서는 [Flutter built-in Kotlin 앱 개발자 마이그레이션](https://docs.flutter.dev/release/breaking-changes/migrate-to-built-in-kotlin/for-app-developers), [Flutter 마이그레이션 개요](https://docs.flutter.dev/release/breaking-changes/migrate-to-built-in-kotlin), [Android built-in Kotlin 전환 가이드](https://developer.android.com/build/migrate-to-built-in-kotlin)다. 문서와 AGP 동작은 2026-10-03에 확인했다.
