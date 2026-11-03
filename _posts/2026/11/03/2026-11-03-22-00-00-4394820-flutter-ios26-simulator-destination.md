---
layout: post
title: "Flutter iOS 26 시뮬레이터 오류 - Unable to find a destination 해결 조건"
description: "Flutter iOS 26에서 Unable to find a destination 오류가 날 때 Xcode 런타임, 시뮬레이터 ID, Flutter 대상 선택을 분리해 점검하는 방법을 정리한다."
date: 2026-11-03
tags: [Flutter, iOS, Xcode, 디버깅, 출시]
comments: true
share: true
---

![Flutter iOS 26 시뮬레이터 destination 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter 앱이 실기기에서는 실행되는데 iOS 26 시뮬레이터에서 `Unable to find a destination matching the provided destination specifier`가 나오면, Dart 코드보다 시뮬레이터 런타임과 대상 ID를 확인해야 한다. iOS 26 시뮬레이터로 개발해야 하는 경우에 유효하며, 실기기 서명 오류나 App Store 업로드 거절을 해결하는 방법은 아니다.

## 오류가 생기는 조건

Flutter 이슈 #180147에는 Flutter 3.38.5와 Xcode 26.0.1에서 실기기는 되지만 iOS 26 시뮬레이터만 destination을 찾지 못한 사례가 기록돼 있다. 이 메시지는 앱 빌드 실패처럼 보이지만, 실제로는 Flutter가 전달한 시뮬레이터 ID가 현재 Xcode가 가진 기기·런타임 목록과 맞지 않는 경우가 많다.

| 확인 대상 | 정상 기준 | 어긋났을 때 나타나는 현상 |
|---|---|---|
| iOS 26 런타임 | Xcode에 설치됨 | iOS 26 기기가 목록에 없음 |
| 시뮬레이터 상태 | `Booted` 또는 사용 가능 | 예전 UDID를 계속 사용 |
| Flutter 대상 | `flutter devices`에 표시됨 | Xcode와 Flutter 목록이 다름 |
| 실제 코드 | 대상 선택 후 컴파일 | destination 이후에만 빌드 오류 |

## 런타임과 ID를 분리해 점검한다

Flutter 공식 iOS 설정 문서는 Xcode의 iOS 플랫폼과 최신 시뮬레이터 런타임을 `xcodebuild -downloadPlatform iOS`로 설치하도록 안내한다. 설치 후 아래 명령으로 현재 목록을 확인한다.

```bash
xcode-select -p
xcodebuild -version
xcrun simctl list runtimes
xcrun simctl list devices available
flutter devices
```

`iOS 26.x (com.apple.CoreSimulator.SimRuntime.iOS-26-...)`가 런타임 목록에 없으면 Xcode를 바꿔도 대상이 생기지 않는다. Xcode 26을 선택한 뒤 런타임을 설치한다.

```bash
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch
xcodebuild -downloadPlatform iOS
```

그 다음 `flutter devices`에 표시된 새 ID를 그대로 지정한다. 예전에 저장한 `--device-id`나 IDE의 오래된 실행 구성이 남아 있으면 새 시뮬레이터를 선택하지 못할 수 있다.

```bash
flutter run -d <flutter_devices에_표시된_시뮬레이터_ID>
```

## 그래도 안 될 때의 출시 전 체크리스트

- Xcode 앱과 `xcode-select -p`가 같은 설치 경로를 가리키는가
- `xcrun simctl list runtimes`에 iOS 26 런타임이 있는가
- `flutter devices`의 기기 ID로 직접 실행했는가
- Xcode에서 같은 시뮬레이터를 선택해 빈 Flutter 프로젝트도 실행되는가
- 특정 프로젝트만 실패하면 `ios/Runner.xcworkspace`를 열어 Scheme과 빌드 구성을 확인했는가

빈 프로젝트도 같은 destination 오류를 내면 플러그인이나 앱 코드보다 Xcode 환경 문제로 범위를 좁힐 수 있다. 반대로 빈 프로젝트는 실행되고 기존 앱만 실패하면, 오래된 실행 설정이나 프로젝트의 iOS 설정을 비교해야 한다. 실제 원인을 확인하지 않은 상태에서 `flutter clean`만 반복하는 것은 런타임과 대상 ID를 고치지 못한다.

2026년 11월 3일 기준으로 확인한 공식 근거는 [Flutter iOS 개발 환경 설정](https://docs.flutter.dev/platform-integration/ios/setup), [Flutter 이슈 #180147](https://github.com/flutter/flutter/issues/180147), [Apple Xcode 26 릴리스 노트](https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes)다. 핵심은 iOS 배포 타깃을 무작정 올리는 것이 아니라, Xcode가 iOS 26 런타임과 현재 시뮬레이터 ID를 함께 인식하는지 확인하는 데 있다.
