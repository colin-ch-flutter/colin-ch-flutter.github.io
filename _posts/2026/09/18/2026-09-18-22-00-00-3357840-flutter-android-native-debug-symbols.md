---
layout: post
title: "Flutter Android App Bundle 네이티브 디버그 심볼 경고 - Play Console 크래시 복구 설정"
description: "Flutter Android 출시 때 표시되는 native debug symbols 경고를 Dart 심볼과 구분하고, AAB와 APK별 Gradle 설정·검증·보관 기준을 정리한다."
date: 2026-09-18
tags: [Flutter, Android, CI/CD, 성능최적화]
comments: true
share: true
---

![Flutter Android App Bundle 네이티브 디버그 심볼 설정](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Play Console에 `This App Bundle contains native code, and you've not uploaded debug symbols` 경고가 보이는 Flutter Android 앱이라면, AAB를 계속 올릴 수는 있어도 운영 크래시의 함수명과 위치를 잃을 수 있다. Play 배포용 AAB라면 `SYMBOL_TABLE`부터 켜는 선택이 현실적이고, 파일·라인까지 필요한 팀만 `FULL`을 선택하면 된다. 반대로 Dart 코드 난독화만 켰다고 이 경고가 해결되지는 않는다.

그림에서 볼 부분은 앱 번들 자체보다, 출시 후 크래시를 사람이 읽을 정보로 되돌리는 별도 심볼 흐름이다.

## 경고가 생기는 이유

Flutter 앱의 release 빌드에는 Dart 코드뿐 아니라 `libflutter.so`와 플러그인의 네이티브 라이브러리도 들어간다. Android Gradle Plugin은 기본적으로 네이티브 라이브러리에서 심볼과 디버그 정보를 제거한다. 그래서 Play Console은 업로드된 번들의 네이티브 크래시를 받더라도 `libflutter.so + 0x...`처럼 해석하기 어려운 스택을 보여줄 수 있다.

여기서 이름이 비슷한 두 심볼을 분리해야 한다.

| 대상 | 만드는 방법 | 쓰이는 곳 |
| --- | --- | --- |
| Dart 심볼 | `--obfuscate --split-debug-info=...` | 난독화된 Dart 스택 복원 |
| 네이티브 심볼 | Gradle `debugSymbolLevel` | Play Console Android vitals의 `.so` 크래시 |

Flutter 공식 문서의 `--split-debug-info`는 Dart 난독화 심볼이다. Play Console의 native 경고에는 Android Gradle 설정이 별도로 필요하다.

## AAB라면 release 변형에 설정한다

`android/app/build.gradle.kts`를 쓰는 프로젝트는 `release` 빌드 타입에 아래 설정을 추가한다. 파일이 Groovy 형식이면 등가 문법을 사용하면 된다.

```kotlin
android {
    buildTypes {
        release {
            ndk {
                debugSymbolLevel = "symbol_table"
            }
        }
    }
}
```

함수명만으로도 충분한 운영 환경은 `symbol_table`이 적합하다. 소스 파일과 라인까지 Play Console에서 보고 싶다면 `full`로 바꾼다.

```kotlin
android {
    buildTypes {
        release {
            ndk {
                debugSymbolLevel = "full"
            }
        }
    }
}
```

Groovy 기반 `android/app/build.gradle`에서는 다음처럼 쓴다.

```groovy
android {
    buildTypes {
        release {
            ndk {
                debugSymbolLevel 'SYMBOL_TABLE'
            }
        }
    }
}
```

설정 후에는 같은 release 변형으로 AAB를 다시 만든다. 이미 올린 번들에 나중에 심볼만 붙이는 것보다, 빌드 산출물과 심볼의 버전·ABI가 일치하는지 관리하기 쉽다.

```bash
flutter clean
flutter pub get
flutter build appbundle --release
```

## APK와 AAB를 혼동하지 않는다

Android 공식 문서 기준으로 AGP 4.1 이상에서 AAB는 심볼 파일을 번들에 포함시킬 수 있다. 반면 APK는 별도 ZIP이 생성되므로 Play Console의 해당 버전에 직접 올려야 한다.

```text
android/app/build/outputs/native-debug-symbols/release/native-debug-symbols.zip
```

실제 경로의 `release`는 flavor를 쓰면 `stagingRelease`처럼 달라진다. APK를 올렸는데 AAB용 자동 포함을 기대하는 경우, 경고가 계속 남는 것이 정상이다.

## 출시 전 확인 체크리스트

- `flutter build appbundle --release`가 실제로 사용하는 변형에 설정했는가
- `SYMBOL_TABLE`로 충분한지, 파일·라인이 필요한지 결정했는가
- `--split-debug-info`로 만든 Dart 심볼과 네이티브 심볼을 같은 파일로 착각하지 않았는가
- flavor·ABI·versionCode가 Play Console에 올릴 산출물과 일치하는가
- `FULL`을 선택했다면 심볼 파일 용량이 조직의 업로드 제한 안에 있는가
- 난독화했다면 해당 빌드의 Dart `*.symbols` 파일을 CI 아티팩트로 보관했는가

`FULL`은 진단 정보가 더 많지만 빌드 산출물과 보관 비용이 커진다. 처음부터 무조건 `FULL`을 켜기보다, 운영팀이 Play Console에서 파일·라인까지 바로 확인해야 하는지로 선택하는 편이 낫다. 어떤 레벨을 쓰든 원본 AAB, versionCode, 커밋 ID, Dart 심볼 파일을 한 묶음으로 보관해야 재현이 가능하다.

짧게 정리하면, 이 경고의 해결책은 Flutter 명령줄 옵션이 아니라 Android release 변형의 `ndk.debugSymbolLevel`이다. AAB는 번들 자동 포함, APK는 생성된 ZIP 수동 업로드라는 차이도 함께 기억해야 한다.

참고 문서:

- [Android Developers: Include native symbols in your release build](https://developer.android.com/build/include-native-symbols)
- [Flutter: Obfuscate Dart code](https://docs.flutter.dev/deployment/obfuscate)
- [Flutter: Build and release an Android app](https://docs.flutter.dev/deployment/android)
