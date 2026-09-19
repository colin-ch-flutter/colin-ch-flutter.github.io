---
layout: post
title: "Flutter iOS Crashlytics Missing dSYM 오류 - TestFlight 출시 후 심볼 업로드 복구"
description: "Flutter iOS 앱을 TestFlight에 올린 뒤 Firebase Crashlytics에 Missing dSYM이 남을 때, 아카이브 UUID 확인부터 수동 업로드와 재발 방지 설정까지 정리한다."
date: 2026-09-19
tags: [Flutter, iOS, Firebase, CI/CD, 배포·운영]
comments: true
share: true
---

![Flutter iOS TestFlight와 Crashlytics dSYM 업로드 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 부분은 IPA 업로드와 Crashlytics 심볼 업로드가 같은 작업이 아니라는 점이다.

Flutter iOS 앱을 TestFlight에 배포한 뒤 Firebase Crashlytics의 dSYMs 탭에 `Missing`이 보인다면, 앱을 다시 빌드하기 전에 해당 빌드의 `.xcarchive`를 보존했는지 확인하는 편이 빠르다. 아카이브가 있으면 UUID가 맞는 dSYM만 수동 업로드하면 되고, 아카이브가 없으면 같은 소스라도 이미 배포된 빌드의 크래시를 복원하지 못할 수 있다. Firebase 초기화 문제나 앱 크래시 자체를 해결하는 글은 아니므로, 여기서는 “출시는 성공했지만 심볼만 빠진 경우”에 적용한다.

## Missing dSYM을 앱 오류로 오해하지 않는다

Release 빌드의 디버그 심볼은 앱 바이너리와 분리된 `.dSYM` 파일에 들어간다. TestFlight에 IPA가 정상 설치돼도 심볼 업로드가 실패하면 Crashlytics에는 주소만 남거나 `(Missing)` 프레임이 표시된다. Firebase 공식 문서도 Xcode Build Phase의 Crashlytics 스크립트, Release dSYM 생성 여부, 수동 업로드를 별도 점검 항목으로 둔다.

| 화면에서 보이는 상태 | 실제로 확인할 대상 | 대응 |
| --- | --- | --- |
| TestFlight 설치·실행 성공, Crashlytics `Missing dSYM` | 해당 App ID와 빌드 UUID | 아카이브의 dSYM 수동 업로드 |
| Archive 안에 dSYM 자체가 없음 | Xcode `Debug Information Format` | Release를 `DWARF with dSYM File`로 수정 후 재배포 |
| dSYM은 있는데 UUID가 다름 | 다른 빌드의 archive를 사용했는지 | App Store Connect 빌드 번호와 UUID를 다시 대조 |
| 업로드 후에도 일부 프레임이 비어 있음 | 플러그인·Flutter 엔진 심볼 누락 | 해당 UUID가 요구된 파일만 추가 업로드 |

## UUID부터 맞춘다

Firebase Console의 Crashlytics에서 앱을 열고 `dSYMs` 탭으로 이동한다. 누락 목록의 UUID를 복사한 뒤 App Store Connect에 올린 당시의 `.xcarchive`에서 같은 UUID를 찾는다. 파일 이름만 보고 “아마 이 빌드겠지”라고 고르면 안 된다.

아카이브 경로는 Flutter 공식 iOS 배포 흐름의 기본값인 `build/ios/archive/앱이름.xcarchive`를 기준으로 잡을 수 있다. 아래 명령은 실제 업로드 전에 UUID만 비교한다.

```bash
ARCHIVE="build/ios/archive/Runner.xcarchive"

find "$ARCHIVE/dSYMs" -name "*.dSYM" -print0 \\
  | xargs -0 -n1 dwarfdump --uuid
```

출력된 UUID가 Firebase의 누락 UUID와 같아야 한다. `Runner.app.dSYM`이라는 이름이 같아도 빌드 번호가 다르면 다른 파일이다. CI에서 archive를 만들었다면 로컬 Mac의 새 archive가 아니라 CI 아티팩트를 내려받아 확인한다.

## 수동 업로드로 이미 배포한 빌드를 복구한다

Firebase 문서가 안내하는 `upload-symbols` 스크립트에 해당 dSYM 경로를 넘기면 된다. CocoaPods를 사용하는 일반적인 Flutter iOS 프로젝트라면 아래처럼 실행할 수 있다.

```bash
PODS_ROOT="ios/Pods"
GSP="ios/Runner/GoogleService-Info.plist"
DSYM="build/ios/archive/Runner.xcarchive/dSYMs/Runner.app.dSYM"

"$PODS_ROOT/FirebaseCrashlytics/upload-symbols" \\
  -gsp "$GSP" \\
  -p ios \\
  "$DSYM"
```

출력에 `Successfully uploaded Crashlytics symbols`가 표시되는지 확인하고, Firebase Console의 dSYMs 목록에서 해당 UUID가 `Uploaded`로 바뀌는지 확인한다. 콘솔 반영에는 시간이 걸릴 수 있다. 이미 발생한 크래시가 즉시 전부 재표시되는 것은 아니며, Firebase는 누락 심볼을 올려도 과거 스택이 모두 자동으로 복원된다고 보장하지 않는다.

Swift Package Manager나 CI에서 CocoaPods 경로가 다르면 스크립트 경로를 고정하지 않는다. Xcode 프로젝트에서 FirebaseCrashlytics 패키지의 `upload-symbols` 위치를 찾고, 같은 `GoogleService-Info.plist`와 UUID가 일치하는 dSYM을 전달해야 한다. 중요한 값은 도구 이름보다 Firebase 앱 ID, 플랫폼, UUID의 일치다.

## 재발 방지는 Build Phase와 보관 정책으로 나눈다

Xcode의 Runner target에서 Release 설정을 확인한다.

```text
Build Settings
└─ Debug Information Format
   └─ Release: DWARF with dSYM File
```

그 뒤 Build Phases에 Firebase Crashlytics run script가 있는지 확인한다. FlutterFire가 생성한 스크립트를 쓰는 경우에도 스크립트가 실제 Archive의 dSYM 경로를 받고 있는지 빌드 로그에서 확인해야 한다. 스크립트가 있어도 CI 권한, 네트워크, Xcode 업데이트 때문에 업로드가 멈출 수 있다. Archive 성공을 심볼 업로드 성공으로 간주하면 이 지점을 놓친다.

출시 파이프라인에는 아래 검사를 남기는 편이 안전하다.

- [ ] App Store Connect 빌드 번호와 archive 빌드 번호가 같다.
- [ ] Release 산출물에 `*.app.dSYM`이 존재한다.
- [ ] `dwarfdump --uuid` 결과가 Crashlytics 누락 UUID와 같다.
- [ ] Crashlytics Build Phase 로그에 업로드 성공이 남는다.
- [ ] `.xcarchive`를 빌드 번호별로 보관한다.

Flutter 공식 문서도 `flutter build ipa`가 archive와 IPA를 만들며, 배포한 각 빌드의 archive를 추적해야 한다고 안내한다. 실제 운영에서 비용이 드는 것은 dSYM 파일 자체보다, 잘못된 UUID를 재빌드로 덮으려다 같은 문제를 반복하는 시간이다. 따라서 수동 업로드가 끝나도 CI 아티팩트 보관 기간과 파일명을 빌드 번호 기준으로 정하는 것이 해결의 일부다.

짧게 정리하면 `Missing dSYM`은 TestFlight 설치 실패가 아니라 크래시 해석 정보의 누락이다. Firebase 누락 UUID를 기준으로 archive의 UUID를 대조하고, 일치하는 dSYM만 업로드한다. archive에 dSYM이 없을 때만 Release 설정과 재배포를 검토하며, 이후에는 Build Phase 로그와 archive 보관을 출시 체크리스트에 포함한다.

참고 문서: [Firebase Crashlytics dSYM 문제 해결](https://firebase.google.com/docs/crashlytics/troubleshooting), [Flutter iOS 배포](https://docs.flutter.dev/deployment/ios), [Apple dSYM 안내](https://developer.apple.com/documentation/xcode/building-your-app-to-include-debugging-information)
