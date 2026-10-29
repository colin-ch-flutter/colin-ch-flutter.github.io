---
layout: post
title: "Flutter iOS 로컬 네트워크 권한 오류 - Debug Bonjour를 Release에 넣지 않는 조건"
description: "Flutter iOS에서 hot reload용 로컬 네트워크 권한이 TestFlight·App Store 출시 설정에 섞이는 원인과 Info-Debug.plist·Info-Release.plist 분리 방법을 정리한다."
date: 2026-10-29
tags: [Flutter, iOS, 출시, 접근성, 플랫폼통합]
comments: true
share: true
---

![Flutter iOS Debug와 Release 네트워크 권한 분리](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter iOS 앱에서 `flutter attach`와 hot reload를 쓰려고 넣은 Bonjour 설정은 Debug에만 둬야 한다. TestFlight나 App Store 빌드에서 `_dartVmService._tcp`를 그대로 유지하면 로컬 네트워크 권한 문구가 출시 앱에 나타나거나 심사 리스크가 생긴다. 반대로 앱 기능 자체가 같은 Wi-Fi의 기기를 검색한다면 Release에서도 로컬 네트워크 권한을 선언해야 하므로, “Release에서 네트워크 키를 전부 삭제”하는 방식도 정답은 아니다.

## 어떤 설정이 문제를 만드는가

Flutter 공식 문서는 iOS 14 이상에서 Dart VM 서비스용 multicast DNS를 Debug에만 활성화하라고 안내한다. 이 서비스는 개발 도구 연결을 위한 것이지 사용자가 쓰는 앱 기능이 아니다. `NSLocalNetworkUsageDescription`은 권한 설명이고, `NSBonjourServices`는 검색할 Bonjour 서비스 목록이라 역할도 다르다.

| 대상 | `NSBonjourServices` | 로컬 네트워크 설명 | 판단 |
|---|---|---|---|
| Debug·Profile | `_dartVmService._tcp` | 개발용 문구 | Flutter 도구 연결에 사용 |
| Release, 로컬 네트워크 미사용 | 없음 | 없음 | 개발 권한을 출시물에서 제거 |
| Release, 기기 검색 기능 사용 | 실제 서비스 타입만 | 사용자에게 기능 설명 | 앱 기능에 필요한 권한 |

여기서 헷갈린 부분은 `Info.plist` 하나를 모든 빌드가 공유한다는 점이다. Debug에서 권한 대화상자가 잘 뜬다는 이유로 그 파일을 Release에도 연결하면, 출시 앱의 권한 목록이 개발 환경과 같아진다.

## 빌드 구성별 plist를 분리한다

Xcode의 Build Settings에서 Debug는 `Info-Debug.plist`, Release는 `Info-Release.plist`를 가리키게 만든다. 아래처럼 개발용 Bonjour 서비스는 Debug 파일에만 둔다.

```xml
<!-- Info-Debug.plist -->
<key>NSBonjourServices</key>
<array>
    <string>_dartVmService._tcp</string>
</array>
<key>NSLocalNetworkUsageDescription</key>
<string>개발 도구 연결을 위해 로컬 네트워크에 접근한다.</string>
```

```text
// Debug configuration
INFOPLIST_FILE = Runner/Info-Debug.plist

// Release configuration
INFOPLIST_FILE = Runner/Info-Release.plist
```

`Info-Release.plist`에는 앱이 실제로 로컬 기기 검색을 하지 않는다면 두 키를 넣지 않는다. 스마트홈·BLE 보조 검색·프린터 탐색처럼 Release에서도 Bonjour를 쓰는 앱이라면 `_http._tcp`처럼 실제 서비스 타입만 남기고, 사용자에게 왜 필요한지 설명하는 문구를 작성한다. 개발용 `_dartVmService._tcp`를 기능용 서비스처럼 재사용하면 안 된다.

## Release 산출물에서 확인할 것

`flutter build ipa --release`가 성공했다고 설정이 올바른 것은 아니다. Archive 안의 최종 앱 번들을 확인해야 한다.

```bash
plutil -p build/ios/archive/Runner.xcarchive/Products/Applications/Runner.app/Info.plist \
  | rg 'NSBonjourServices|NSLocalNetworkUsageDescription|dartVmService'
```

출력에 `_dartVmService._tcp`가 나오면 Release plist 경로, Xcode의 `INFOPLIST_FILE`, Build Phases에 복사된 plist를 함께 확인한다. `Info-Release.plist`를 프로젝트에 추가했지만 Copy Bundle Resources에만 넣고 빌드 설정을 바꾸지 않은 경우도 있다. Archive를 다시 만들 때는 이전 산출물을 혼동하지 않도록 `flutter clean` 후 실행한다.

## 출시 전 체크리스트

- [ ] Debug와 Release가 서로 다른 `INFOPLIST_FILE`을 가리킨다.
- [ ] Release 앱 기능이 로컬 네트워크를 쓰지 않으면 Bonjour 키가 없다.
- [ ] 로컬 네트워크를 쓰는 경우 실제 서비스 타입과 사용자 설명만 선언했다.
- [ ] Archive의 `Runner.app/Info.plist`에서 `_dartVmService._tcp`가 빠졌다.
- [ ] 권한 허용·거부 상태에서 TestFlight 기능을 각각 확인했다.

Flutter 공식 기준은 Debug에서 Dart multicast DNS를 켜고 Release에서는 켜지 않는 것이다. Apple도 로컬 네트워크를 실제로 사용하는 앱에는 `NSLocalNetworkUsageDescription`을 요구한다. 따라서 해결 기준은 키를 무조건 삭제하는 것이 아니라, 개발 도구 권한과 제품 기능 권한을 빌드 구성 및 서비스 타입별로 분리하는 데 있다.

참고: [Flutter iOS 프로젝트 설정](https://docs.flutter.dev/add-to-app/ios/project-setup), [Flutter iOS 디버깅](https://docs.flutter.dev/platform-integration/ios/ios-debugging), [Apple NSLocalNetworkUsageDescription](https://developer.apple.com/documentation/bundleresources/information-property-list/nslocalnetworkusagedescription)
