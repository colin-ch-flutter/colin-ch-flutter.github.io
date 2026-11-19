---
layout: post
title: "Flutter iOS 27 푸시·딥링크 오류 - UIScene 전환 뒤 AppDelegate 콜백이 사라질 때"
description: "Flutter 3.41과 Xcode 27로 iOS 앱을 출시한 뒤 푸시 탭과 딥링크가 조용히 무시되면 AppDelegate와 FlutterSceneDelegate의 책임을 나누고 콜백을 검증하는 방법을 정리한다."
date: 2026-11-20
tags: [Flutter, iOS, 출시, Firebase, CI/CD]
comments: true
share: true
---
![Flutter iOS 27 UIScene 전환 뒤 푸시와 딥링크를 점검하는 화면](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

이 문제는 Flutter 3.41 이상과 Xcode 27로 빌드하면서 `AppDelegate`에 푸시·딥링크 코드를 직접 넣은 앱에 해당한다. 앱이 실행은 되지만 알림을 눌러도 특정 화면으로 가지 않거나 Universal Link가 홈 화면으로만 열리면 `Info.plist`에 Scene 설정을 복사하는 것보다 콜백의 소유자를 나누는 게 우선이다. 기본 `AppDelegate`를 수정하지 않은 프로젝트는 Flutter의 자동 이전 대상이므로 이 글의 코드를 무작정 추가하지 않는다.

## 왜 실행은 되는데 이벤트만 사라지나

Xcode 27 SDK에서는 UIKit 앱이 `UIScene` 수명 주기를 사용해야 한다. Flutter 공식 문서 기준으로 Flutter 3.41부터 기본 iOS 앱은 자동 이전되지만, 커스텀 `AppDelegate`가 있으면 UI 이벤트가 더 이상 예전 application 콜백으로 오지 않을 수 있다. 특히 `application(_:didFinishLaunchingWithOptions:)`의 launch options만 읽는 딥링크 처리와 `window`를 직접 만지는 코드가 취약하다.

| 확인한 현상 | 실제로 갈린 책임 | 우선 확인할 곳 |
|---|---|---|
| 앱 시작 후 바로 종료 | Scene 미이전 또는 `window` 강제 캐스팅 | `AppDelegate`, `Info.plist` |
| 푸시 탭은 열리지만 목적지 없음 | 알림 응답 처리를 UI 수명 주기에만 연결 | 알림 플러그인 설정과 앱 라우터 |
| Universal Link가 Safari로 감 | `scene(_:continue:)` 전달 누락 또는 Associated Domains | `SceneDelegate`, AASA 파일 |
| 일반 실행은 정상 | 프로세스 이벤트와 UI 이벤트가 다른 경로 | AppDelegate와 SceneDelegate를 따로 로그 |

## 플러그인 등록 위치를 바꾼다

커스텀 `AppDelegate`에서 `GeneratedPluginRegistrant`를 예전 launch 메서드에만 호출하고 있다면 암시적 Flutter 엔진 초기화 콜백으로 옮긴다. 이 코드는 플러그인 등록이 Scene 연결보다 늦어져 `MissingPluginException`이 나는 앱에서 확인할 기준점이다.

```swift
import Flutter
import UIKit

@main
@objc class AppDelegate: FlutterAppDelegate, FlutterImplicitEngineDelegate {
  func didInitializeImplicitFlutterEngine(
    _ engineBridge: FlutterImplicitEngineBridge
  ) {
    GeneratedPluginRegistrant.register(
      with: engineBridge.pluginRegistry
    )
  }
}
```

실제 Flutter SDK의 프로토콜·브리지 타입이 프로젝트 버전과 다르면 IDE 자동 완성으로 선언을 확인해야 한다. 핵심은 메서드 이름을 외워 복사하는 데 있지 않고, 등록을 `didFinishLaunching`에만 묶지 않는 데 있다. Flutter 공식 예제는 현재 SDK에서 `FlutterImplicitEngineDelegate`와 `FlutterImplicitEngineBridge`를 기준으로 설명한다.

## 푸시와 딥링크는 각각 로그를 남긴다

알림 탭과 Universal Link를 하나의 “앱 열림” 로그로 합치면 원인을 잃는다. 앱이 cold start인지 background 복귀인지도 함께 기록하고, 수신한 URL 또는 notification payload를 라우터에 넘기기 전 한 번만 출력한다. 로그는 `kDebugMode`로 감싸지 않아 TestFlight에서도 확인할 수 있게 한다.

```swift
func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
  guard let url = userActivity.webpageURL else { return }
  print("[universal-link] \(url.absoluteString)")
  // Flutter 라우터로 전달할 pending URL을 저장한다.
}
```

`firebase_messaging`이나 `app_links`를 쓰는 경우에는 플러그인이 Scene 콜백을 지원하는 버전인지 확인한다. 앱에서 같은 이벤트를 직접 받아 플러그인에도 넘기면 cold start에서 두 번 라우팅될 수 있다. 하나의 진입점을 정하고, 로그인 전에는 pending route로 보관한 뒤 인증 완료 시 한 번만 소비하는 편이 안전하다.

## 출시 전 체크리스트

- [ ] `flutter build ios` 출력에 자동 UIScene 이전 성공 여부가 남는다.
- [ ] 커스텀 `AppDelegate`가 `window`나 `rootViewController`를 launch 콜백에서 강제 사용하지 않는다.
- [ ] `GeneratedPluginRegistrant` 등록 위치가 현재 Flutter SDK의 implicit engine 방식과 맞다.
- [ ] 앱이 종료된 상태에서 푸시 탭, 백그라운드에서 푸시 탭, 실행 중 푸시 탭을 각각 시험했다.
- [ ] 실기기에서 Notes의 Universal Link를 눌렀고, TestFlight 서명 앱으로도 확인했다.
- [ ] 앱 삭제 후 재설치하여 예전 링크 캐시와 pending route가 결과를 오염시키지 않는지 확인했다.

핵심은 iOS 27 크래시와 이벤트 누락을 같은 문제로 취급하지 않는 것이다. Flutter 3.41의 기본 프로젝트는 자동 이전될 수 있지만, 커스텀 수명 주기·푸시 SDK·딥링크 플러그인을 넣은 앱은 AppDelegate가 프로세스 이벤트를 맡고 SceneDelegate가 UI 이벤트를 전달하도록 나눠야 한다. 공식 기준은 [Flutter의 UIScene 이전 안내](https://docs.flutter.dev/release/breaking-changes/uiscenedelegate)와 [Flutter iOS add-to-app의 SceneDelegate 설명](https://docs.flutter.dev/add-to-app/ios/add-flutter-screen)에서 확인할 수 있다.
