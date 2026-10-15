---
layout: post
title: "Flutter 3.44 iOS SwiftPM 빌드 오류 - Release.xcconfig의 CocoaPods include 제거 조건"
description: "Flutter 3.44에서 iOS Swift Package Manager 전환 뒤 Archive만 실패한다면 Release.xcconfig의 오래된 CocoaPods include와 플러그인 호환성을 확인하는 복구 순서를 정리한다."
date: 2026-10-16
tags: [Flutter, iOS, CI/CD, 테스트]
comments: true
share: true
---

![Flutter iOS Swift Package Manager release 빌드 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter 3.44 이상에서 iOS 앱을 Swift Package Manager(SwiftPM)로 전환한 뒤 `flutter run`은 되는데 `flutter build ipa --release`나 Xcode Archive만 실패한다면, `ios/Flutter/Release.xcconfig`에 남은 CocoaPods 경로부터 확인하는 선택이 유리하다. 모든 플러그인이 아직 SwiftPM을 지원하지 않는 프로젝트라면 CocoaPods를 바로 삭제하지 말고, 두 방식을 함께 쓰는 조건부터 분리해야 한다.

## 증상은 플러그인 오류처럼 보인다

실패 로그에는 `module 'Flutter' not found`, `Unable to find module dependency`, `could not build module`처럼 플러그인 이름이 먼저 나온다. 하지만 원인은 플러그인 코드가 아니라 Release 설정이 예전 Pods 산출물을 계속 가리키는 경우가 있다. Debug와 Release가 서로 다른 `.xcconfig`를 읽기 때문에 개발 실행만 통과하는 것이다.

Flutter 공식 문서 기준으로 3.44부터 SwiftPM 지원이 기본 활성화되며, SwiftPM으로 완전히 옮긴 앱은 `Debug.xcconfig`와 `Release.xcconfig`에서 `Pods/Target Support Files`를 가리키는 `#include`를 제거해야 한다. 다만 SwiftPM을 지원하지 않는 의존성이 하나라도 있으면 Flutter는 CocoaPods로 되돌아갈 수 있다.

| 프로젝트 상태 | `Pods` include 처리 | CocoaPods 파일 삭제 |
|---|---|---|
| 모든 플러그인이 SwiftPM 지원 | Debug·Release에서 제거 | 가능, 삭제 전 Archive 확인 |
| 일부 플러그인만 지원 | 해당 플러그인 때문에 유지 | 삭제하면 안 됨 |
| 전환 중이며 원인 분리 필요 | 한 번에 삭제하지 않음 | 백업 후 단계적으로 진행 |

## Release 설정에서 먼저 볼 부분

오래된 설정이 남았는지 확인하는 명령이다. `Pods-Runner.release.xcconfig`를 무조건 지우는 것이 아니라, 현재 프로젝트가 CocoaPods를 실제로 사용하는지부터 판단한다.

```bash
rg -n "Pods/Target Support Files|\.xcconfig" \
  ios/Flutter ios/Runner.xcodeproj
```

SwiftPM만 사용하는 프로젝트라면 `ios/Flutter/Debug.xcconfig`, `ios/Flutter/Release.xcconfig`, 필요하면 `Profile.xcconfig`에서 다음과 같은 줄을 제거한다.

```xcconfig
// SwiftPM 완전 전환 앱에서는 남아 있으면 안 되는 예시
#include? "Pods/Target Support Files/Pods-Runner/Pods-Runner.release.xcconfig"
#include "Generated.xcconfig"
```

`Generated.xcconfig`까지 지우면 안 된다. 이 파일은 Flutter 빌드 설정을 전달하는 역할이므로, 제거 대상은 CocoaPods를 가리키는 include다. 변경 뒤에는 Xcode에서 `ios/Runner.xcworkspace`를 열고 `Runner` 타깃의 Package Dependencies와 `FlutterGeneratedPluginSwiftPackage`가 연결됐는지 확인한다.

## CocoaPods를 함께 써야 하는 경우

광고·지도·네이티브 인증 플러그인처럼 아직 SwiftPM을 지원하지 않는 패키지가 있으면 Pods 경로를 지우는 방식이 정답이 아니다. 이때는 플러그인별 지원 상태를 확인하고, SwiftPM으로 자동 전환된 패키지와 CocoaPods로 남은 패키지의 빌드 설정이 같은 Scheme에 포함되는지 확인해야 한다.

Flutter 프로젝트의 `pubspec.yaml`에서 SwiftPM을 잠시 끄는 것도 원인 분리에는 쓸 수 있다.

```yaml
flutter:
  config:
    enable-swift-package-manager: false
```

이 설정은 최종 해결책이 아니다. SwiftPM을 끈 뒤 Archive가 통과한다면 “플러그인이 고장 났다”가 아니라 패키지 매니저 전환 또는 Xcode 프로젝트 연결이 문제였다는 뜻이다. 그 상태로 출시하기보다 문제를 일으킨 플러그인을 찾고, 지원 버전으로 올리거나 CocoaPods 유지 조건을 문서화해야 한다.

## 재현과 출시 전 체크리스트

Debug 실행만으로는 이 문제를 검증할 수 없다. 아래 순서로 Release 산출물까지 확인한다.

- [ ] `flutter --version`이 3.44 이상인지 기록했다.
- [ ] `Release.xcconfig`에 남은 `Pods/Target Support Files` include를 확인했다.
- [ ] `Generated.xcconfig`는 유지했다.
- [ ] 모든 플러그인이 SwiftPM을 지원하는지 확인했다.
- [ ] `flutter clean && flutter pub get` 후 `flutter build ipa --release`를 실행했다.
- [ ] Xcode Archive에서 실제 Runner 타깃과 패키지 의존성을 확인했다.

내가 확인한 기준일은 2026년 10월 16일이다. [Flutter Swift Package Manager 앱 개발자 문서](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers)는 3.44부터의 기본 활성화, CocoaPods fallback, `Debug.xcconfig`·`Release.xcconfig`의 Pods include 제거와 clean/build 검증을 안내한다. [Flutter 3.38 릴리스 문서](https://flutter.dev/blog/whats-new-in-flutter-3-38)는 최신 iOS·Xcode 지원 흐름도 설명한다.

핵심은 `flutter clean`을 반복하는 데 있지 않다. SwiftPM을 완전히 쓰는 앱인지, CocoaPods를 함께 써야 하는 앱인지 먼저 나누고, 그 판단에 맞춰 `.xcconfig`와 Xcode 타깃 연결을 정리해야 Release Archive가 재현 가능해진다.
