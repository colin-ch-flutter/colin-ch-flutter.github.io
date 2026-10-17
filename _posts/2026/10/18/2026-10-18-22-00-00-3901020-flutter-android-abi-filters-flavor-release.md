---
layout: post
title: "Flutter Android 3.35 abiFilters 오류 - flavor release에서 ABI가 무시될 때"
description: "Flutter 3.35 이후 Android flavor와 release APK에서 abiFilters가 예상과 다르게 적용되는 조건, libflutter.so 누락을 막는 빌드 명령과 확인 절차를 정리한다."
date: 2026-10-18
tags: [Flutter, Android, 배포, CI/CD, 성능최적화]
comments: true
share: true
---

![Flutter Android release APK의 ABI 필터와 flavor 빌드 점검](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

이 그림에서는 앱 기능보다 release 산출물에 어떤 CPU 아키텍처의 네이티브 라이브러리가 들어가는지가 핵심이다.

Flutter 3.35 이상에서 Android flavor별로 `abiFilters`를 직접 관리하는 앱이라면, `build.gradle`만 고쳐서는 release APK의 ABI가 바뀌지 않을 수 있다. `flutter build apk --release --flavor prod`가 정상 종료돼도 특정 기기에서 `libflutter.so`를 찾지 못하거나, 예상보다 큰 APK가 나올 때가 대상이다. ABI를 직접 제한할 필요가 없는 일반 앱은 설정을 추가하지 않는 편이 안전하다.

## Flutter 3.35에서 달라진 조건

Flutter Gradle Plugin은 3.35부터 debuggable이 아닌 빌드에 기본 ABI 필터를 넣는다. 기본 목록은 `armeabi-v7a`, `arm64-v8a`, `x86_64`다. Google Play가 Flutter가 실제로 지원하지 않는 x86 앱까지 배포 대상으로 표시해 `libflutter.so` 로딩 크래시가 나는 문제를 줄이려는 변경이다.

| 상황 | 우선 선택 | 확인할 위험 |
| --- | --- | --- |
| ABI를 따로 나누지 않는 일반 AAB | 기본값 유지 | 별도 수정 불필요 |
| 기기별 APK가 필요함 | `--split-per-abi` | APK마다 version code가 달라짐 |
| `release` 또는 flavor마다 직접 필터 지정 | 기본 필터 비활성화 후 직접 관리 | 플래그를 빼면 설정이 무시될 수 있음 |
| 네이티브 라이브러리가 없는 순수 Dart 앱 | ABI 설정을 건드리지 않음 | 불필요한 Gradle 충돌 |

## flavor release에서 무시될 때

`buildTypes.release.ndk.abiFilters`나 `productFlavors.prod.ndk.abiFilters`에 의존하는 프로젝트라면 Flutter가 넣은 기본 필터가 먼저 적용되는지 확인해야 한다. 공식 마이그레이션 문서는 이런 사용자 지정 구성을 유지해야 할 때 `disable-abi-filtering` 속성을 빌드 명령에 전달하는 경로를 제시한다.

아래 명령은 `prod` flavor의 자체 ABI 설정을 사용해야 하는 경우에만 적용한다.

```bash
flutter build apk \
  --release \
  --flavor prod \
  -Pdisable-abi-filtering=true
```

이 방식은 Flutter의 기본 필터를 끄는 것이지, ABI를 자동으로 올바르게 골라주는 기능이 아니다. `arm64-v8a`만 넣은 AAR을 `armeabi-v7a` 기기에 배포하면 앱이 설치되더라도 네이티브 로딩에서 실패할 수 있다. flavor 이름, 실제 지원 기기, 플러그인이 제공하는 `.so` 목록을 하나의 표로 관리해야 한다.

기기별 APK가 목적이면 수동 `abiFilters`보다 Flutter가 제공하는 분할 빌드가 단순하다.

```bash
flutter build apk --release --split-per-abi
```

단일 APK를 만들어야 하고 앱의 `defaultConfig`에서 ABI를 고정하는 구조라면, 공식 예시처럼 기존 집합을 비운 뒤 필요한 ABI만 추가한다.

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters.clear()
            abiFilters.addAll(listOf("arm64-v8a", "x86_64"))
        }
    }
}
```

## 산출물에서 확인할 것

`flutter run` 성공은 출시 APK 검증이 아니다. release 파일을 만든 뒤 Android Studio APK Analyzer 또는 압축 해제 결과에서 `lib/<ABI>/libflutter.so`, `lib/<ABI>/libapp.so`, 플러그인의 `.so`가 같은 ABI 집합으로 존재하는지 확인한다. `x86_64` 에뮬레이터만 통과하고 arm64 실기기에서 실패하는 식의 오판을 막으려면 최소 한 대의 실제 출시 ABI로 설치해야 한다.

CI에는 아래 조건을 체크리스트로 남긴다.

- [ ] Flutter SDK 버전과 `disable-abi-filtering` 사용 여부를 기록했다.
- [ ] flavor별 `applicationId`와 지원 ABI를 표로 고정했다.
- [ ] release APK 내부에 `libflutter.so`와 `libapp.so`가 같은 ABI 경로에 있다.
- [ ] 네이티브 플러그인의 `.so`가 앱이 지원하는 모든 ABI를 제공한다.
- [ ] Play에 올릴 AAB는 bundletool 또는 Play Console의 기기별 APK로 확인했다.
- [ ] `arm64-v8a` 실기기와 `x86_64` 테스트 환경을 각각 한 번씩 실행했다.

핵심은 `abiFilters`를 무조건 늘리거나 줄이는 것이 아니다. 커스텀 flavor 구성이 없다면 Flutter 기본값을 유지하고, 구성이 있다면 기본 필터를 끈 뒤 앱이 실제로 제공할 ABI와 플러그인 바이너리를 함께 검증해야 한다. 이 문제 때문에 별도 유료 빌드 서비스를 구매할 필요는 없으며, APK Analyzer와 `--split-per-abi`만으로 1차 판별할 수 있다.

공식 기준은 [Flutter의 Android 기본 abiFilters 변경 문서](https://docs.flutter.dev/release/breaking-changes/default-abi-filters-android), [Flutter Android 배포 가이드](https://docs.flutter.dev/deployment/android), 그리고 `productFlavors`에서 필터가 적용되지 않았던 [Flutter 이슈 #175845](https://github.com/flutter/flutter/issues/175845)에서 확인했다.
