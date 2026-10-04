---
layout: post
title: "Flutter Firebase Crashlytics 난독화 오류 - release 스택을 복구하는 심볼 업로드 조건"
description: "Flutter release에서 --obfuscate와 --split-debug-info를 사용한 뒤 Firebase Crashlytics 스택이 읽히지 않을 때 Firebase App ID, 심볼 보관, CI 업로드를 점검하는 방법을 정리한다."
date: 2026-10-05
tags: [Flutter, Firebase, 테스트, CI/CD, 성능최적화]
comments: true
share: true
---

![Flutter Firebase Crashlytics release 심볼 업로드 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

Flutter 앱을 `--obfuscate --split-debug-info`로 출시했다면 Firebase Crashlytics에 난독화된 Dart 스택만 보일 때 **같은 빌드에서 나온 심볼 파일을 Firebase App ID에 업로드하는 선택**이 맞다. 난독화를 끄면 당장 읽을 수 있지만, 이미 배포한 빌드의 복구에는 도움이 되지 않는다. 반대로 난독화를 쓰지 않는 내부 테스트 앱이라면 이 절차를 추가할 필요가 없다.

## `dSYM`과 Dart 심볼을 혼동하면 안 된다

Crashlytics에서 보이는 심볼은 플랫폼과 빌드 옵션에 따라 역할이 나뉜다.

| 증상 | 확인할 파일 | 해결 경로 |
| --- | --- | --- |
| Dart 함수명이 `a`, `b`처럼 보임 | `app.android-arm64.symbols` 등 | `firebase crashlytics:symbols:upload` |
| iOS native 프레임이 주소만 보임 | dSYM | Xcode `upload-symbols` 단계 |
| Android `.so` 프레임이 주소만 보임 | native debug symbols | Gradle `debugSymbolLevel` |

`--split-debug-info`가 만드는 `*.symbols`는 Dart 난독화 매핑 파일이다. 이 파일을 잃으면 이후 스택을 복원할 수 없으므로 CI 아티팩트나 접근 제한 저장소에 보관해야 한다.

## 출시 빌드와 업로드를 같은 규칙으로 묶기

아래 예시는 Android App Bundle을 만들면서 빌드 번호별 디렉터리에 심볼을 남기는 형태다. 경로를 고정하면 CI에서 이전 빌드 심볼을 잘못 업로드하는 실수를 줄일 수 있다.

```bash
set -euo pipefail

BUILD_ID="${GITHUB_RUN_NUMBER:-local}"
SYMBOL_DIR="build/symbols/${BUILD_ID}"

flutter build appbundle --release \
  --obfuscate \
  --split-debug-info="${SYMBOL_DIR}"

firebase crashlytics:symbols:upload \
  --app="${FIREBASE_ANDROID_APP_ID}" \
  "${SYMBOL_DIR}"
```

`FIREBASE_ANDROID_APP_ID`는 패키지명 `com.example.app`이 아니다. `google-services.json`의 `mobilesdk_app_id` 또는 Firebase 프로젝트 설정의 Android 앱 ID를 넣는다. Firebase CLI 11.9.0 이상을 기준으로 확인한다.

## 업로드가 성공해도 스택이 안 풀리는 조건

가장 흔한 원인은 파일은 존재하지만 다른 빌드의 파일이라는 점이다. `versionCode`나 ABI가 다른 심볼을 올리면 명령이 성공해도 이벤트와 매칭되지 않는다.

```text
[ ] --obfuscate와 --split-debug-info를 release 명령에 함께 기록했다
[ ] 심볼 디렉터리를 빌드 번호별로 분리했다
[ ] 해당 빌드의 AAB와 *.symbols를 같은 아티팩트 묶음으로 보관했다
[ ] 패키지명이 아닌 Firebase App ID를 사용했다
[ ] 심볼 업로드가 앱 크래시 보고보다 먼저 실행된다
[ ] Firebase 콘솔에서 새 이벤트의 Dart 프레임이 읽히는지 확인했다
```

Firebase 문서 기준으로 Flutter 3.12.0 이상과 `firebase_crashlytics` 3.3.4 이상이면 Flutter 심볼 자동 처리 경로를 사용할 수 있다. Android 설정이 오래됐다면 `flutterfire configure` 뒤 Crashlytics 플러그인 적용 여부도 확인한다. Apple 플랫폼은 Dart 심볼과 별개로 dSYM 업로드 단계가 필요하다.

별도 모니터링 서비스를 구매하지 않아도 해결할 수 있다. 심볼 파일은 소스 코드처럼 접근을 제한하고, 배포 버전별로 덮어쓰지 않는다. 난독화는 비밀값 암호화가 아니므로 앱에 API 키를 넣어도 안전해지지 않는다.

짧게 정리하면 `Crashlytics 이벤트 → 플랫폼별 심볼 종류 확인 → 같은 빌드의 파일 업로드` 순서다. Dart 스택이면 Firebase App ID와 `--split-debug-info` 경로를, iOS native 스택이면 dSYM을, Android `.so` 스택이면 Gradle native symbols를 따로 점검해야 한다.

- [Firebase Crashlytics Flutter: 읽을 수 있는 크래시 보고서](https://firebase.google.com/docs/crashlytics/flutter/get-deobfuscated-reports)
- [Flutter: Dart 코드 난독화](https://docs.flutter.dev/deployment/obfuscate)
