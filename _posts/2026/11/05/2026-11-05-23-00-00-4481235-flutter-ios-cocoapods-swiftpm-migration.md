---
layout: post
title: "Flutter iOS CocoaPods 전환 오류 - SwiftPM 마이그레이션 전 플러그인 점검"
description: "Flutter 3.44 이후 iOS 프로젝트에서 CocoaPods를 바로 지워도 되는 조건과 2026년 12월 2일 registry read-only 전환 전에 SwiftPM 지원 플러그인과 Xcode 설정을 확인하는 방법을 정리한다."
date: 2026-11-05
tags: [Flutter, iOS, SwiftPM, CI/CD, 출시]
comments: true
share: true
---

![Flutter iOS CocoaPods에서 Swift Package Manager로 전환하는 Xcode 프로젝트](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter 3.44 이상에서 iOS 앱을 운영한다면 CocoaPods를 오늘 당장 삭제할지보다 플러그인별 Swift Package Manager(SwiftPM) 지원 여부를 먼저 확인하는 선택이 유리하다. 모든 의존성이 SwiftPM을 지원하면 CocoaPods를 제거해도 되지만, 하나라도 미지원이면 Flutter가 CocoaPods fallback을 사용한다. 2026년 12월 2일 CocoaPods registry가 read-only로 바뀌는 시점은 기존 빌드가 즉시 깨지는 날이 아니라 새 pod 버전을 배포할 수 없게 되는 기준일이다.

## 문제는 `pod install`보다 플러그인 조합이다

Flutter 공식 문서에 따르면 3.44부터 SwiftPM이 기본 활성화되고, SwiftPM을 지원하지 않는 의존성은 CocoaPods로 fallback된다. 그래서 `flutter run`이 되는 것만으로 전환이 끝났다고 보면 안 된다. CI의 새 Mac, Release Archive, 특정 flavor에서만 CocoaPods 경로가 다시 살아날 수 있다.

| 프로젝트 상태 | CocoaPods 제거 | 출시 판단 |
|---|---:|---|
| 모든 iOS 플러그인이 SwiftPM 지원 | 가능 | Archive와 CI 재현 후 전환 |
| 일부 플러그인만 미지원 | 불가 | SwiftPM + CocoaPods 혼용 유지 |
| Podfile에 직접 추가한 SDK가 있음 | 자동 제거 금지 | 수동 Package 지원 또는 대체 검토 |

표에서 봐야 할 부분은 `SwiftPM 기본값`과 `CocoaPods fallback`이 동시에 존재할 수 있다는 점이다.

## 전환 전 확인할 명령

먼저 `pubspec.yaml`의 플러그인 목록과 iOS 네이티브 의존성을 기록한다. 실제 사용 중인 버전과 플러그인 저장소의 `Package.swift` 지원 여부를 함께 봐야 한다.

```bash
flutter --version
flutter pub deps --style=compact
rg -n "pod |target|use_frameworks|enable-swift-package-manager" ios pubspec.yaml
```

Flutter 3.44 이상에서 SwiftPM을 다시 켜야 한다면 아래 설정을 사용한다. 개인 컴퓨터 전역 설정만 바꾸지 말고 팀 프로젝트의 `pubspec.yaml`과 CI 이미지도 같은 기준으로 맞춘다.

```bash
flutter config --enable-swift-package-manager
flutter pub get
flutter run
```

Xcode에서는 `Runner.xcworkspace`를 열고 `FlutterGeneratedPluginSwiftPackage`가 Runner target dependency로 연결됐는지, Scheme의 Build Pre-actions에 `Run Prepare Flutter Framework Script`가 있는지 확인한다. flavor가 있다면 각 Scheme를 따로 확인해야 한다.

## CocoaPods를 지워도 되는 경우

모든 의존성이 SwiftPM으로 이동했고 Podfile에 직접 추가한 SDK도 없다면 Flutter 공식 절차대로 iOS 프로젝트에서 CocoaPods 흔적을 정리할 수 있다. 이 작업은 되돌리기 번거로우므로 현재 브랜치를 보존한 뒤 진행한다.

```bash
cd ios
pod deintegrate
cd ..

rm -rf ios/Pods ios/.symlinks
rm -f ios/Podfile ios/Podfile.lock
```

`rm`은 위 경로처럼 프로젝트 내부의 확인된 대상에만 적용한다. `ios/Flutter/Debug.xcconfig`와 `Release.xcconfig`에 `Pods/Target Support Files`를 가리키는 `#include`가 남아 있으면 제거한다. `Generated.xcconfig`까지 같이 지우면 Flutter 빌드 설정이 사라질 수 있으니 대상이 아니다.

정리 뒤에는 의존성 설치와 실제 출시 빌드를 분리해서 확인한다.

```bash
flutter clean
flutter pub get
flutter build ipa --release
```

반대로 광고·지도·결제 SDK 중 하나라도 SwiftPM을 지원하지 않거나 Podfile에 수동 설정이 있다면 CocoaPods를 유지하는 편이 안전하다. `flutter config --no-enable-swift-package-manager`는 원인 분리용 임시 우회일 뿐, 2026년 12월 이후의 장기 해법은 아니다.

## 출시 전 체크리스트

- [ ] `pubspec.lock` 기준으로 iOS 플러그인 목록과 SwiftPM 지원 여부를 기록했다.
- [ ] Xcode의 `FlutterGeneratedPluginSwiftPackage`와 모든 flavor Scheme를 확인했다.
- [ ] Podfile에 Flutter 플러그인 외 수동 SDK가 있는지 확인했다.
- [ ] CocoaPods를 제거하기 전 `flutter build ipa --release`를 별도 브랜치에서 통과시켰다.
- [ ] CI의 새 환경에서도 `flutter pub get`과 Archive가 같은 방식으로 동작한다.

현재 기준으로 CocoaPods registry의 read-only 전환은 기존 앱을 즉시 중단시키는 정책이 아니다. 다만 미지원 플러그인이 새 버전을 CocoaPods에만 올리면 업데이트 선택지가 줄어든다. CocoaPods를 무조건 삭제하거나 SwiftPM을 무조건 끄기보다, 플러그인 지원 상태와 CI Archive 결과가 모두 확인된 경우에만 전환을 승인하는 기준이 현실적이다.

참고 기준은 2026년 11월 5일 확인한 [Flutter Swift Package Manager 앱 개발자 문서](https://docs.flutter.dev/packages-and-plugins/swift-package-manager/for-app-developers), [Flutter의 CocoaPods 전환 안내](https://flutter.dev/blog/saying-goodbye-to-cocoapods-swift-package-manager-is-soon-the-default-in-flutter), [flutter-webrtc의 SwiftPM 지원 이슈](https://github.com/flutter-webrtc/flutter-webrtc/issues/2045)다.
