---
layout: post
title: "Flutter Android Release 빌드 오류 - coreLibraryDesugaring을 minSdk와 구분하는 조건"
description: "Flutter Android Release에서 D8 Invoke-customs와 core library desugaring 오류가 날 때, AGP·minSdk·Gradle 설정을 구분해 수정하고 출시 전 검증하는 방법을 정리한다."
date: 2026-10-30
tags: [Flutter, Android, Firebase, CI/CD]
comments: true
share: true
---

![Flutter Android Release 빌드에서 Java API 호환성을 점검하는 화면](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter Android 앱에서 `D8: Invoke-customs are only supported starting with Android O` 또는 `The dependency contains Java 8 bytecode`가 Release 빌드에서만 보인다면 `minSdk`를 무작정 올리기보다 core library desugaring을 확인하는 편이 유리하다. 오래된 Android Gradle Plugin(AGP) 프로젝트나 Firebase SDK를 추가한 프로젝트가 대상이다. 반대로 앱이 이미 최신 AGP를 쓰고 있고 오류가 Kotlin 컴파일, JDK 버전, 네이티브 라이브러리 충돌이라면 이 설정만으로 해결되지 않는다.

## 오류가 생기는 조건

Java 8 언어 문법과 `java.time`, `java.util.stream` 같은 Java API는 서로 다른 문제다. Flutter 플러그인이나 Firebase Android SDK가 이런 API를 포함하면 D8이 오래된 Android에서도 실행할 수 있게 변환해야 한다. Firebase 공식 FAQ는 AGP 4.2 이하에서 Java 8 지원이 꺼져 있으면 이 오류가 발생한다고 설명한다. 확인 날짜는 2026년 10월 30일이다.

| 확인 항목 | 판단 | 대응 |
|---|---|---|
| AGP 4.0 이상 | Java API desugaring 지원 가능 | `coreLibraryDesugaring` 활성화 |
| `minSdk` 25 이하 유지 필요 | 구형 기기 호환이 제품 조건 | minSdk를 유지하고 변환 라이브러리 사용 |
| minSdk 26 이상으로 올릴 수 있음 | Android O 미만 지원 불필요 | 오류 메시지처럼 minSdk 상향도 선택지 |
| `debug`만 성공하고 `release` 실패 | Release variant의 Gradle 설정 누락 가능 | app 모듈의 `compileOptions`와 dependency 확인 |

## Flutter Android 프로젝트 설정

Kotlin DSL을 사용하는 최근 Flutter 프로젝트라면 앱 모듈에 아래처럼 Java 호환성과 desugaring dependency를 같은 모듈에 둔다. 이 코드는 Java API 변환이 필요한 이유가 있어야 적용한다.

```kotlin
// android/app/build.gradle.kts
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
        isCoreLibraryDesugaringEnabled = true
    }
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.3")
}
```

Groovy DSL이면 `isCoreLibraryDesugaringEnabled = true` 대신 `coreLibraryDesugaringEnabled true`를 쓴다. `desugar_jdk_libs` 버전은 AGP와 함께 맞춰야 하므로 공식 Android 문서의 호환 예시를 기준으로 프로젝트 버전을 확인한다. 최신 Flutter 템플릿이 Java 17을 요구하는 환경이라면 source·target을 17로 맞출 수 있지만, 핵심은 `isCoreLibraryDesugaringEnabled`와 전용 dependency가 빠지지 않는 것이다.

`coreLibraryDesugaring`을 일반 `implementation`에 넣으면 D8 변환 단계가 해당 라이브러리를 찾지 못한다. 반대로 dependency만 추가하고 `compileOptions`를 빼도 같은 오류가 남는다. 두 설정은 한 쌍으로 점검해야 한다.

## minSdk를 올리는 선택이 항상 정답은 아니다

Firebase 공식 안내에는 `minSdkVersion`을 26 이상으로 올리는 대안도 있다. 그러나 이 변경은 빌드 오류만 없애는 설정이 아니라 Android 7.1 이하 사용자를 지원 대상에서 제외하는 출시 결정이다. 설치 기반이 넓거나 이미 배포된 앱이면 desugaring을 유지하는 비용과 지원 기기 감소 비용을 비교해야 한다.

```bash
flutter clean
flutter pub get
flutter build appbundle --release
```

빌드가 성공해도 실제 기기 검증을 생략하면 안 된다. `minSdk`가 낮은 테스트 기기에서 날짜·컬렉션 API를 호출하는 화면을 열고, Firebase 초기화와 알림 같은 플러그인 경로를 Release 서명 변형으로 확인한다. CI에서는 `android/app/build.gradle` 또는 `.kts` 변경을 캐시 키에 반영하고, 오래된 Gradle 캐시만 지워서 해결됐다고 판단하지 않는다.

## 출시 전 체크리스트

- [ ] 오류 로그에 `Invoke-customs`, `Java 8 bytecode`, `desugaring`이 실제로 포함되는지 확인했다.
- [ ] 설정을 프로젝트 모듈이 아닌 `android/app` 모듈에 넣었다.
- [ ] `compileOptions`와 `coreLibraryDesugaring(...)`을 함께 적용했다.
- [ ] `minSdk` 상향이 지원 기기 정책을 바꾸는지 검토했다.
- [ ] 캐시를 비운 Debug가 아니라 Release AAB와 낮은 API 기기에서 플러그인 경로를 확인했다.

핵심은 `minSdk`와 desugaring을 같은 해결책으로 취급하지 않는 데 있다. 구형 Android 지원을 유지해야 하면 변환 설정을 맞추고, 지원 대상에서 API 26 미만을 명시적으로 제외할 수 있을 때만 minSdk 상향을 선택한다. 참고: [Android Java 8+ API desugaring 문서](https://developer.android.com/studio/write/java8-support), [Firebase Android 오류 FAQ](https://firebase.google.com/docs/android/troubleshooting-faq).
