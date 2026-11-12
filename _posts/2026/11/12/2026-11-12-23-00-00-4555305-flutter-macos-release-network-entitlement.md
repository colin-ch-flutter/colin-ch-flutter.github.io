---
layout: post
title: "Flutter macOS release 네트워크 오류 - debug에서만 localhost 연결이 될 때"
description: "Flutter macOS 앱이 debug에서는 API나 localhost에 연결되지만 release에서 Operation not permitted를 낼 때, client·server entitlement를 구분하고 서명된 앱까지 확인하는 절차를 정리한다."
date: 2026-11-12
tags: [Flutter, macOS, 배포·운영, CI/CD]
comments: true
share: true
---
![Flutter macOS release 앱의 네트워크 entitlement 점검](https://images.unsplash.com/photo-1517336714739-489689fd1ca8?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 부분은 연결 코드가 아니라 release 앱에 서명된 네트워크 권한이다.

Flutter macOS 앱이 debug에서는 API와 localhost에 연결되는데 `flutter build macos --release` 결과에서 `SocketException: Operation not permitted`가 난다면 HTTP 패키지부터 바꿀 문제가 아니다. 외부 API 호출은 `com.apple.security.network.client`, 다른 프로세스나 로컬 서버가 앱으로 들어오는 연결은 `com.apple.security.network.server`가 대상이다. 일반적인 외부 API 클라이언트라면 `client`만 필요하고, 로컬 daemon·WebSocket 서버를 앱이 받아야 할 때만 `server`를 추가한다. 파일 접근 글의 해결책처럼 App Sandbox를 통째로 끄는 선택은 배포 앱에 적합하지 않다.

## 증상으로 entitlement를 나눈다

| 상황 | 확인할 키 | 흔한 오해 |
| --- | --- | --- |
| `https://api.example.com` 요청 실패 | `network.client` | Android INTERNET 권한을 찾음 |
| `127.0.0.1`의 서버로 연결 실패 | 보통 `network.client` | localhost면 권한이 없다고 생각함 |
| 앱이 포트를 열고 외부 연결을 받음 | `network.server` | Debug 설정이 Release에도 복사됐다고 생각함 |

Flutter 공식 문서 기준으로 `network.server`는 Flutter 도구와 실행 중인 앱의 통신을 위해 Debug/Profile에 기본으로 들어간다. 그래서 로컬 서버 기능은 개발 중 정상이어도 release에서만 막힐 수 있다.

## Release entitlement에 필요한 키만 넣는다

외부 API 호출과 로컬 포트 수신이 모두 필요한 경우의 예다. 하나만 필요하면 나머지는 넣지 않는다.

```xml
<key>com.apple.security.app-sandbox</key>
<true/>
<key>com.apple.security.network.client</key>
<true/>
<key>com.apple.security.network.server</key>
<true/>
```

파일은 `macos/Runner/Release.entitlements`에 둔다. Xcode의 Signing & Capabilities에서 capability를 켰더라도 실제 Release 타깃이 이 파일을 사용하는지 확인해야 한다. 수정 뒤에는 이전 `.app`을 재사용하지 않도록 clean build를 하고, 실행 중인 앱을 완전히 종료한 뒤 다시 실행한다.

## 최종 앱의 서명을 확인한다

설정 파일만 보지 말고 생성된 앱에 권한이 실제로 들어갔는지 확인한다.

```bash
APP="build/macos/Build/Products/Release/Runner.app"
codesign -d --entitlements :- "$APP" 2>/dev/null
flutter clean
flutter build macos --release
```

키가 없으면 Xcode 프로젝트의 Release Code Sign 설정과 파일 경로를 다시 본다. 키가 있는데 외부 API만 실패하면 인증서·프록시·DNS 문제를 별도로 확인한다. `network.server`는 포트 충돌이나 서버 실행까지 해결하지 않는다.

## 출시 전 체크리스트

- [ ] 외부 API인지, 앱이 포트를 열어 받는 구조인지 구분했다.
- [ ] `Release.entitlements`에 필요한 키만 넣었다.
- [ ] `codesign`으로 최종 `.app`의 entitlement를 확인했다.
- [ ] Debug가 아닌 release 앱에서 API·localhost 연결을 다시 확인했다.
- [ ] App Sandbox를 끄는 방식으로 문제를 숨기지 않았다.

macOS Flutter 공식 안내는 네트워크 권한을 `*.entitlements`에서 설정하고, incoming 연결이 필요한 앱은 `Runner-Release.entitlements`에도 `network.server`를 넣어야 한다고 설명한다. 2026년 11월 12일 확인 기준으로, 핵심은 debug 동작 여부가 아니라 배포 산출물에 권한이 실제 서명됐는지다.

참고: [Flutter macOS 앱 빌드 안내](https://docs.flutter.dev/platform-integration/macos/building), [Flutter 네트워킹 공식 문서](https://docs.flutter.dev/data-and-backend/networking), [Apple App Sandbox와 entitlement](https://developer.apple.com/documentation/security/app_sandbox)
