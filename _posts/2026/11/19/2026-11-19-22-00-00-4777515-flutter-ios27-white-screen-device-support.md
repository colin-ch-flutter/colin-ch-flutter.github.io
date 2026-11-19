---
layout: post
title: "Flutter iOS 27 흰 화면 오류 - 실기기 Device Support 심볼 준비와 flutter run 복구"
description: "Flutter iOS 27 실기기에서 flutter run 뒤 흰 화면이 오래 지속될 때 Xcode Device Support 심볼을 준비하고, Flutter 버전별 복구 조건을 확인하는 방법을 정리한다."
date: 2026-11-19
tags: [Flutter, iOS, Xcode, 디버깅, 출시]
comments: true
share: true
---

![Flutter iOS 27 실기기 흰 화면과 Device Support 준비](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

iOS 27로 업데이트했거나 새 iPhone을 페어링한 뒤 `flutter run`이 흰 화면에서 멈춘다면 앱 코드보다 Mac의 Xcode Device Support 심볼 준비 상태를 먼저 확인하는 편이 맞다. Xcode 27과 Flutter 3.50 조합이면 CLI가 준비를 자동 처리하지만, Flutter가 낮거나 준비가 실패한 환경은 `xcodebuild -prepareDeviceSupport`를 직접 실행해야 한다. TestFlight 출시 앱의 흰 화면에는 이 해결책이 적용되지 않는다.

## 증상이 앱 오류처럼 보이는 이유

Flutter 공식 이슈에는 iOS 27 실기기에서 앱이 설치·실행된 뒤 흰 화면으로 멈추고, LLDB가 기기의 shared cache를 찾지 못한다는 경고를 남기는 사례가 기록돼 있다. 새 OS의 디버깅 심볼을 Xcode가 내려받는 동안 Flutter가 Dart VM Service 연결을 기다리는 상황이다. 약 5분 뒤 열릴 수도 있지만, 매번 기다리는 방식은 정상적인 개발 조건이 아니다.

| 확인 지점 | Device Support 문제 | 앱 초기화 문제 |
|---|---|---|
| 대상 | 새로 업데이트·페어링한 실기기 | 특정 앱 또는 모든 기기 |
| 로그 단서 | `libobjc.A.dylib` shared cache 경고 | `main()` 이후 플러그인·네트워크 오류 |
| 우선 조치 | Xcode 심볼 준비 | 앱 초기화와 플러그인 분리 |

## Xcode 심볼을 직접 준비하는 방법

기기의 UDID를 확인한 뒤, Xcode가 설치된 Mac에서 아래 명령을 실행한다. `flutter devices` 출력의 `id` 값을 사용하면 된다.

```bash
flutter devices
xcrun xcodebuild -prepareDeviceSupport -destination "id=기기-UDID"
```

명령이 끝날 때까지 USB 연결을 유지하고 iPhone의 잠금도 해제한다. Xcode가 Developer Mode, 컴퓨터 신뢰, 기기 페어링을 요구하면 화면에서 승인한다. 준비가 끝난 뒤 다시 실행한다.

```bash
flutter clean
flutter pub get
flutter run -d 기기-UDID
```

여기서 `flutter clean`은 심볼을 내려받는 핵심 명령이 아니다. 이전 빌드 산출물이 남아 결과를 헷갈리게 할 때만 함께 사용한다. `prepareDeviceSupport`가 실패했다면 Xcode 선택 경로와 버전을 확인한다.

```bash
xcode-select -p
xcodebuild -version
flutter doctor -v
```

## Flutter 버전에 따른 판단

Flutter 3.50 공식 변경 사항에는 새로 페어링했거나 업데이트한 iPhone에서 CLI가 기기를 자동 준비하고, 준비에 실패하면 안내하도록 바뀐 내용이 포함됐다. 따라서 이 증상이 반복되면 무작정 프로젝트 코드를 고치기보다 `flutter --version`부터 기록한다.

- Flutter 3.50 이상: 자동 준비 로그와 실패 원인을 확인하고, 실패 시 수동 명령을 실행한다.
- 이전 버전: `prepareDeviceSupport`를 직접 실행한 뒤 재현 여부를 비교한다.
- 시뮬레이터만 정상: 실기기 심볼·페어링 문제를 우선 의심한다.
- TestFlight나 App Store 설치본만 흰 화면: 이 글의 디버깅 심볼 문제로 단정하지 않고 `main()` 초기화, Firebase 설정, 네이티브 플러그인 로그를 따로 확인한다.

출시 전 체크는 `새 iOS 기기 연결 → prepareDeviceSupport 완료 → debug 실행 → release 실행 → 백그라운드 복귀` 순서로 남기는 것이 안전하다. 시뮬레이터에서 화면이 열렸다는 사실만으로 실기기 디버깅 환경이 준비됐다고 볼 수 없다.

핵심은 흰 화면이 보인다는 결과가 아니라, 앱이 첫 프레임을 못 그린 것인지 Flutter가 디버거 연결을 기다리는 것인지 나누는 데 있다. iOS 27 실기기에서만 오래 멈추고 shared cache 경고가 보이면 Device Support를 준비한 뒤 다시 판단하면 된다.

참고 문서: [Flutter 3.50 변경 사항](https://flutter.dev/blog/whats-new-in-flutter-3-50), [Flutter iOS Device Support 관련 이슈](https://github.com/flutter/flutter/issues/189284), [Flutter iOS 개발 환경 설정](https://docs.flutter.dev/platform-integration/ios/setup)
