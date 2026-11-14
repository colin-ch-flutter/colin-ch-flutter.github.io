---
layout: post
title: "Flutter iOS SwiftPM Archive 오류 - 플러그인 번들에 프로비저닝 프로파일이 전파될 때"
description: "Flutter 3.44 이상 iOS Swift Package Manager 전환 뒤 Archive에서 플러그인 번들이 프로비저닝 프로파일을 지원하지 않는다는 오류가 날 때, 전역 수동 서명 설정을 Runner로 좁히는 방법을 정리한다."
date: 2026-11-14
tags: [Flutter, iOS, CI/CD, 배포, SwiftPM]
comments: true
share: true
---

![Flutter iOS Swift Package Manager Archive 서명 오류를 점검하는 Xcode 화면](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

Flutter 3.44 이상에서 iOS Swift Package Manager(SwiftPM)를 쓰고, `flutter run`은 되는데 Archive에서 `does not support provisioning profiles`가 나온다면 프로파일을 플러그인 번들까지 전달하는 설정을 먼저 걷어내야 한다. 앱을 수동 서명해야 하는 경우에도 프로파일의 적용 대상은 `Runner`와 앱 확장으로 한정한다. SwiftPM을 쓰지 않거나 자동 서명을 쓰는 프로젝트에는 이 오류의 해결책을 그대로 적용할 필요가 없다.

## 왜 플러그인 번들이 프로파일을 요구하는가

Flutter 3.44부터 SwiftPM이 iOS·macOS 의존성 관리의 기본 경로가 됐다. 리소스를 포함한 플러그인은 코드 타깃과 별도의 `.bundle` 타깃으로 Archive에 나타날 수 있다. 이 번들은 앱처럼 프로비저닝 프로파일을 발급받는 대상이 아니다.

| 로그의 단서 | 실제 대상 | 판단 |
|---|---|---|
| `Runner`가 프로파일을 찾지 못함 | 앱 타깃 | Bundle ID·팀·인증서 확인 |
| `url_launcher_ios_url_launcher_ios` 등 | SwiftPM 플러그인 타깃 | 전역 수동 서명 설정 제거 |
| `.../Flutter/ephemeral/Packages/.packages/` | Flutter가 생성한 패키지 사본 | 생성 파일을 직접 수정하지 않음 |

특히 오래된 CI 명령의 `PROVISIONING_PROFILE_SPECIFIER=...`, `CODE_SIGN_IDENTITY=...` 또는 Release용 공통 `.xcconfig`가 모든 빌드 타깃에 적용되면 이 현상이 나타난다. CocoaPods의 `post_install`에서 `CODE_SIGNING_ALLOWED = NO`를 넣어 우연히 통과하던 프로젝트가 SwiftPM 전환 뒤 깨지는 경우도 같은 계열이다.

## 수정 순서

오류 경로가 `Runner`가 아니라 플러그인 타깃인지 확인한 뒤, 저장소에서 전역 서명 값을 찾는다.

```sh
rg -n "PROVISIONING_PROFILE|CODE_SIGN_IDENTITY|CODE_SIGN_STYLE|DEVELOPMENT_TEAM" ios .github fastlane
```

공통 `Release.xcconfig`나 CI의 `xcodebuild` 인자에 프로파일을 넣었다면 삭제하고, Xcode의 `Runner` 타깃 `Signing & Capabilities`에서만 팀과 배포 프로파일을 지정한다. 자동 서명을 허용할 수 있는 CI라면 Runner에 유효한 팀을 설정하고 Archive 단계에서 프로파일을 강제하지 않는 편이 단순하다.

```sh
# 프로파일 이름을 모든 SwiftPM 타깃에 주입하지 않는 Archive 예시
flutter clean
flutter pub get
flutter build ipa --release
```

수동 서명이 꼭 필요하다면 `-xcconfig`에 프로파일 값을 넣은 뒤 전체 워크스페이스에 전달하지 말고, Xcode 프로젝트의 Runner 타깃 설정으로 이동한다. `ios/Flutter/ephemeral` 아래의 `Package.swift`나 생성된 패키지 프로젝트를 고쳐도 다음 빌드에서 다시 생성되므로 복구책이 아니다. SwiftPM을 지원하지 않는 플러그인이 섞였다면 SwiftPM을 끄는 것보다 해당 플러그인의 지원 버전과 CocoaPods fallback 조건을 별도로 확인한다.

## Archive 전에 확인할 체크리스트

- [ ] 오류 대상이 `Runner`인지 SwiftPM 플러그인·리소스 번들인지 분리했다.
- [ ] `Release.xcconfig`, Fastlane, CI 환경 변수에 전역 프로파일 값이 남아 있지 않다.
- [ ] Bundle ID와 프로파일은 앱 타깃에만 연결돼 있다.
- [ ] `flutter clean` 뒤 실제 Archive를 새로 만들었다.
- [ ] `Payload/*.app/Frameworks`와 플러그인 리소스가 포함된 최종 아카이브를 확인했다.
- [ ] 시뮬레이터 빌드 성공이 아니라 `flutter build ipa --release` 결과를 출시 기준으로 삼았다.

이 문제에서 `Set the provisioning profile value to Automatic`이라는 Xcode의 문구를 플러그인 패키지 타깃에 그대로 적용할 수는 없다. 패키지 타깃은 앱 프로젝트에서 직접 관리하는 일반 타깃이 아니기 때문이다. 해결의 핵심은 프로파일을 더 추가하는 것이 아니라, 서명 책임을 `Runner`로 좁히는 데 있다. Flutter의 SwiftPM 기본 전환과 프로젝트별 비활성화 옵션은 [Swift Package Manager for app developers](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers), 배포 명령은 [Flutter iOS 배포 문서](https://docs.flutter.dev/deployment/ios)에서 확인할 수 있다.
