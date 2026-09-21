---
layout: post
title: "Flutter Android R8 Missing Classes 오류 - debug는 되는데 release에서 크래시 나는 이유"
description: "Flutter Android release 빌드에서만 R8 Missing Classes와 런타임 크래시가 발생할 때 missing_rules.txt를 확인하고 필요한 keep rule만 추가하는 복구 절차를 정리한다."
date: 2026-09-21
tags: [Flutter, Android, 배포·운영, 성능최적화, CI/CD]
comments: true
share: true
---

![Flutter Android release 빌드에서 R8 Missing Classes를 점검하는 터미널과 App Bundle](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

그림에서 볼 부분은 debug APK가 아니라 실제 출시 산출물인 release App Bundle을 점검하는 흐름이다.

Flutter Android 앱에서 debug는 정상인데 Play 배포용 release만 시작 직후 종료된다면 R8이 의심 대상이다. Java·Kotlin 클래스를 reflection이나 JNI로 이름을 찾아 쓰는 플러그인을 포함한 앱은 필요한 클래스가 축소·이름 변경으로 사라질 수 있다. 반대로 단순히 `Missing class` 경고가 보였다는 이유만으로 패키지 전체에 `-keep class **`를 넣으면 앱 크기와 난독화 효과를 함께 잃는다.

## 문제 상황을 먼저 나눈다

Flutter 공식 Android 배포 문서 기준으로 release 빌드에는 R8 코드 축소가 적용된다. 그래서 `flutter run`으로 확인한 결과와 `flutter build appbundle --release` 결과가 다를 수 있다.

| 증상 | 먼저 볼 곳 | 의미 |
| --- | --- | --- |
| `assembleRelease`가 `R8: Missing class`로 실패 | Gradle 출력, `missing_rules.txt` | 의존성 또는 keep rule 구성이 부족할 가능성 |
| 빌드는 성공하지만 release 실행 직후 크래시 | Logcat의 `ClassNotFoundException`, `NoSuchMethodException` | reflection/JNI 대상이 제거·난독화됐을 가능성 |
| 특정 기능을 열 때만 크래시 | 해당 Flutter 플러그인의 Android 코드와 consumer rules | 동적 로딩되는 클래스가 기능 실행 시점에 처음 필요해진 경우 |
| debug와 profile은 정상, release만 실패 | release AAB의 실제 설치 테스트 | 축소·난독화 차이를 확인해야 함 |

핵심은 R8을 꺼서 출시하는 것이 아니라, 어떤 클래스가 동적으로 사용되는지 증명하는 것이다. `--no-shrink` 같은 임시 우회는 원인 분리에 참고할 수 있지만, 그 상태를 최종 배포 설정으로 삼으면 문제를 숨긴다.

## `missing_rules.txt`에서 시작한다

R8 오류가 발생한 뒤 생성된 파일을 먼저 확인하면 무작정 규칙을 늘리지 않아도 된다. 프로젝트와 사용하는 Android Gradle Plugin 버전에 따라 세부 경로가 다를 수 있지만 보통 아래 위치에서 찾는다.

```bash
find android build -path '*mapping*release*missing_rules.txt' -print 2>/dev/null
```

파일 안의 클래스가 실제 앱 기능에서 필요한지 확인한다. 예를 들어 사용하지 않는 선택적 SDK의 클래스라면 해당 라이브러리 버전을 올리거나 의존성을 제거하는 편이 낫다. 반대로 플러그인이 `Class.forName()`이나 JNI로 클래스명을 전달한다면 keep rule 후보가 된다.

R8이 제안한 규칙을 그대로 전부 붙이지 말고, 패키지와 호출 경계를 좁혀 `android/app/proguard-rules.pro`에 추가한다.

```proguard
# 예시: 실제 로그에 나온 플러그인 패키지로 범위를 좁힌다.
-keep class com.example.vendor.** { *; }

# reflection으로 이름을 읽는 모델의 생성자만 필요한 경우
-keepclassmembers class com.example.vendor.model.** {
    <init>(...);
}
```

위 패키지명은 예시다. 실제 앱에서는 Logcat과 플러그인 소스의 패키지명을 대조해야 한다. `com.example.**`처럼 앱 전체를 보존하는 규칙은 마지막 수단으로 남긴다. 플러그인 작성자라면 앱에 규칙을 복사시키기보다 `consumer-rules.pro`에 필요한 규칙을 제공하는 편이 재사용에 맞다.

## release 산출물로 복구 여부를 확인한다

규칙을 추가한 뒤에는 debug가 아니라 같은 방식으로 AAB를 만들고, release 서명·설정에 가까운 기기에 설치해 기능을 호출한다. 빌드 성공만으로 reflection 크래시가 해결됐다고 판단하면 안 된다.

```bash
flutter clean
flutter pub get
flutter build appbundle --release

# release APK를 별도로 검증할 때
flutter build apk --release --split-per-abi
adb install -r build/app/outputs/flutter-apk/app-arm64-v8a-release.apk
adb logcat -c
adb shell monkey -p com.example.app 1
adb logcat -d | rg -i 'FATAL EXCEPTION|ClassNotFound|NoSuchMethod|UnsatisfiedLinkError'
```

가상 패키지명과 ABI 파일명은 앱 설정에 맞게 바꾼다. AAB만 Play에 올리는 앱이라면 Play Console의 내부 테스트 트랙에 먼저 업로드해 Google Play가 생성한 split 설치에서도 해당 기능을 실행한다. 로컬 universal APK만 통과한 결과는 충분한 증거가 아니다.

## 출시 전 체크리스트

- [ ] `missing_rules.txt`의 각 클래스가 실제로 필요한지 확인했다.
- [ ] keep 범위를 플러그인·모델 패키지 수준으로 제한했다.
- [ ] debug가 아닌 release APK 또는 내부 테스트 AAB를 설치했다.
- [ ] 로그인, 결제, 푸시, BLE처럼 native 플러그인을 호출하는 경로를 각각 열었다.
- [ ] `ClassNotFoundException`과 `UnsatisfiedLinkError`가 없는지 Logcat으로 확인했다.
- [ ] 난독화를 사용한다면 해당 빌드의 mapping 파일을 CI 아티팩트로 보관했다.

R8 오류의 무료 해결책은 필요 클래스만 정확히 보존하는 것이다. 앱 전체 keep이나 무조건적인 축소 해제는 빠른 임시 진단에는 쓸 수 있어도, 다운로드 크기·난독화·향후 유지비를 키울 수 있다. 내가 확인한 기준으로는 Flutter 공식 release 문서와 Android 공식 keep rule 문서가 공통으로 동적 로딩 코드를 별도 검증 대상으로 본다.

참고한 공식 문서는 [Flutter Android 앱 빌드·배포](https://docs.flutter.dev/deployment/android), [Android R8 keep rule 개요](https://developer.android.com/topic/performance/app-optimization/keep-rules-overview), [Android keep rule 사용 사례](https://developer.android.com/topic/performance/app-optimization/keep-rule-examples)다.
