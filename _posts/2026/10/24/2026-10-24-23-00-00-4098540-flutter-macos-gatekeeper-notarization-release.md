---
layout: post
title: "Flutter macOS release 오류 - ZIP 앱이 열리지 않을 때 Developer ID와 notarization 점검"
description: "Flutter macOS 앱을 ZIP으로 배포한 뒤 다른 Mac에서 열리지 않는다면 ad-hoc 서명과 Gatekeeper를 구분해야 한다. Developer ID, Hardened Runtime, notarytool, staple 검증 순서를 정리한다."
date: 2026-10-24
tags: [Flutter, macOS, release, notarization, CI/CD]
comments: true
share: true
---
![Flutter macOS release 앱의 코드 서명과 notarization 점검](https://images.unsplash.com/photo-1517336714739-489689fd1ca8?auto=format&fit=crop&w=1600&q=80)

Flutter macOS 앱을 내 Mac에서는 열 수 있는데 ZIP으로 전달한 다른 Mac에서 “확인할 수 없는 개발자” 또는 “손상되어 열 수 없음”이 나온다면, 앱 코드보다 배포 서명을 우선 확인해야 한다. Mac App Store 배포가 아니라 웹사이트에서 `.app`이나 DMG를 내려받게 할 경우에는 Developer ID Application으로 서명하고 Apple 공증(notarization) 결과를 붙이는 선택이 유리하다. 사내에서 서명되지 않은 빌드를 직접 실행하는 상황에는 이 절차가 필요하지 않다.

## `flutter run`이 성공해도 배포 앱은 막히는 이유

`flutter build macos --release`는 release 산출물을 만들지만, 이것만으로 외부 배포에 필요한 신뢰 체인이 완성됐다고 볼 수 없다. 개발 Mac에서는 Xcode가 만든 ad-hoc 서명이나 로컬 권한 때문에 실행되지만, 다운로드한 앱은 Gatekeeper가 서명자와 공증 티켓을 다시 확인한다. Flutter 공식 문서도 Mac App Store 밖에서 배포하는 앱은 notarization이 필요하다고 안내한다.

| 확인 대상 | 통과 기준 | 자주 하는 오해 |
|---|---|---|
| 서명 인증서 | `Developer ID Application` | Mac App Distribution 인증서로도 충분하다고 생각한다 |
| Hardened Runtime | Xcode Release 타깃에 활성화 | Sandbox 권한과 같은 설정으로 착각한다 |
| 공증 | `notarytool` 결과가 `Accepted` | 업로드만 하면 공증된 것으로 생각한다 |
| 배포 파일 | 앱 또는 DMG에 ticket을 `staple` | ZIP을 다시 만들면 검증이 끝난다고 생각한다 |

## 산출물과 서명을 분리해서 확인한다

앱을 빌드한 뒤 Finder에서 열어보는 것보다, 실제 배포할 `.app`을 명령으로 검사하는 편이 빠르다. 아래 경로는 기본 Flutter 프로젝트의 Release 위치다.

```bash
flutter build macos --release

APP="build/macos/Build/Products/Release/Runner.app"
codesign --verify --deep --strict --verbose=2 "$APP"
codesign -dv --verbose=4 "$APP" 2>&1 | rg 'Authority|TeamIdentifier|flags'
spctl --assess --type execute --verbose=4 "$APP"
```

출력에 `Authority=Developer ID Application:`이 없거나 `TeamIdentifier`가 비어 있으면 공증 단계로 넘어가도 해결되지 않는다. `spctl`의 `rejected`는 단순히 앱 내부 파일 권한이 아니라 Gatekeeper가 현재 배포 신뢰 정보를 인정하지 않는다는 뜻이다.

## `notarytool`로 공증하고 ticket을 붙인다

Apple Developer 계정과 Developer ID Application 인증서가 준비됐다는 전제에서, 앱을 ZIP으로 묶어 공증 서비스에 제출한다. 앱별 비밀번호를 명령행에 직접 쓰지 말고 CI secret이나 키체인 프로파일을 사용한다.

```bash
ditto -c -k --keepParent "$APP" Runner-macos.zip

xcrun notarytool submit Runner-macos.zip \
  --keychain-profile "AC_NOTARY" \
  --wait

xcrun stapler staple "$APP"
spctl --assess --type execute --verbose=4 "$APP"
```

공증 결과가 `Accepted`여도 사용자가 받을 파일이 공증 전 앱을 담은 ZIP이면 의미가 없다. `stapler staple`은 공증 ticket을 앱에 붙이는 단계이고, 그 뒤에 최종 ZIP 또는 DMG를 만들어야 한다.

```bash
ditto -c -k --keepParent "$APP" Runner-macos.zip
xcrun stapler validate "$APP"
```

DMG를 배포한다면 DMG를 만든 뒤 DMG 자체도 공증 대상으로 삼고, 배포 전에 깨끗한 Mac에서 압축 해제부터 실행까지 확인한다. 앱을 Xcode에서 실행한 결과만으로는 다운로드·격리 속성·Gatekeeper를 검증할 수 없다.

## 출시 체크리스트

- [ ] Release 타깃에 Developer ID Application 서명이 적용됐다.
- [ ] Hardened Runtime이 켜져 있고 필요한 예외만 추가했다.
- [ ] `codesign --verify`와 `spctl --assess`를 최종 `.app`에 실행했다.
- [ ] `notarytool` 결과가 `Accepted`다.
- [ ] 공증 후 `stapler validate`가 통과했다.
- [ ] 공증 후 다시 만든 ZIP 또는 DMG를 다른 Mac에서 실행했다.

핵심은 `build macos` 성공, 코드 서명, Apple 공증, 최종 압축 파일 검증을 서로 다른 단계로 보는 것이다. Mac App Store에만 배포한다면 Developer ID 방식 대신 App Store Connect 배포 절차를 따르면 되고, 외부 ZIP 배포라면 Gatekeeper를 우회하라는 안내보다 서명·Hardened Runtime·공증을 출시 조건으로 고정하는 편이 재현 가능한 해결책이다.

참고 기준은 2026년 10월 24일 Flutter macOS 배포 문서와 Apple의 macOS notarization·외부 배포 문서다. Apple 공증은 App Review가 아니라 자동 검증 절차이므로, 공증 성공과 앱 기능 심사를 같은 의미로 해석하면 안 된다.

참고 문서: [Flutter macOS 앱 빌드와 배포](https://docs.flutter.dev/deployment/macos), [Flutter macOS 빌드 안내](https://docs.flutter.dev/platform-integration/macos/building), [Apple notarization 안내](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution), [Apple 외부 배포 안내](https://help.apple.com/xcode/mac/current/en.lproj/dev033e997ca.html)
