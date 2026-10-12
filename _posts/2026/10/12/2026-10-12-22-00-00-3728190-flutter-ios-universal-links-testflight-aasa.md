---
layout: post
title: "Flutter iOS Universal Links 오류 - TestFlight에서 Safari로 열릴 때 AASA와 서명 확인"
description: "Flutter iOS Universal Links가 debug에서는 동작하지만 TestFlight에서 Safari로 열릴 때 Associated Domains, AASA, Team ID, 캐시를 확인하는 출시 점검 순서다."
date: 2026-10-12
tags: [Flutter, iOS, 배포, 딥링크, 테스트]
comments: true
share: true
---

![Flutter iOS Universal Links TestFlight 출시 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter iOS Universal Links가 debug에서는 앱을 열지만 TestFlight 링크는 Safari로 간다면, Flutter 라우터보다 `Associated Domains` entitlement와 `apple-app-site-association` 파일을 먼저 확인하는 게 맞다. 웹 도메인이 없거나 커스텀 스킴만 필요하다면 이 설정은 필요 없다.

## 실패 지점을 먼저 나눈다

| 증상 | 우선 확인할 곳 | 흔한 원인 |
| --- | --- | --- |
| 링크가 항상 Safari에서 열림 | entitlement와 AASA | 도메인 오타, Team ID·Bundle ID 불일치 |
| debug만 앱으로 열림 | Debug/Release 설정 | Release entitlement에 도메인이 빠짐 |
| 앱은 열리지만 화면이 안 바뀜 | Flutter 링크 수신 코드 | OS 전달은 성공했지만 라우팅을 처리하지 않음 |
| 수정 후에도 이전 결과 유지 | 재설치·Apple CDN | 파일 캐시와 기기 검증 시점 차이 |

## Release entitlement를 먼저 확인한다

Xcode에서 Runner target의 Signing & Capabilities에 Associated Domains를 추가하고, Release에서도 다음 값이 들어가는지 확인한다. `applinks:` 뒤에는 포트나 경로를 붙이지 않는다.

```xml
<key>com.apple.developer.associated-domains</key>
<array>
    <string>applinks:example.com</string>
    <string>applinks:www.example.com</string>
</array>
```

Debug entitlement에만 값을 넣으면 로컬 실행은 되지만 TestFlight archive에는 빠질 수 있다. Archive를 만든 뒤 Xcode의 Signing 정보를 보거나 다음처럼 Release 산출물의 entitlement를 확인한다.

```bash
codesign -d --entitlements :- \
  "build/ios/archive/Runner.xcarchive/Products/Applications/Runner.app"
```

## AASA는 도메인마다 따로 제공한다

`https://example.com/.well-known/apple-app-site-association`에 확장자 없는 JSON을 제공한다. `www.example.com`을 entitlement에 따로 적었다면 그 호스트에도 같은 경로의 파일이 있어야 한다.

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAMID.com.example.app"],
        "components": [{ "/": "/invite/*" }]
      }
    ]
  }
}
```

`TEAMID`는 Apple Developer의 Team ID이고 뒤의 Bundle ID는 실제 bundle identifier다. 브라우저에서 보이는 것만으로 충분하지 않다. 리다이렉트나 로그인 없이 HTTPS로 직접 200 응답을 주고, JSON으로 내려와야 한다.

```bash
curl -i https://example.com/.well-known/apple-app-site-association
```

## TestFlight에서 검증할 순서

1. 기존 앱을 삭제하고 새 TestFlight 빌드를 설치한다.
2. 메일·메시지처럼 다른 앱에서 `https://example.com/invite/abc`를 누른다.
3. Safari로 열리면 `applinks` 호스트와 AASA의 `appIDs`를 비교한다.
4. 앱은 열리는데 화면 이동이 없으면 Flutter의 초기 링크와 백그라운드 링크 처리를 분리해 확인한다.

Safari에서 같은 도메인의 페이지를 보고 있다가 같은 도메인의 링크를 누르면 Safari에 남을 수 있다. 다른 앱에서 눌러야 OS 연결을 제대로 시험할 수 있다. AASA를 고친 직후 모든 기기에 반영된다고 가정해서도 안 된다. Apple CDN과 기기 검증에 시간이 걸린다.

URL 파싱 테스트와 iOS의 앱 선택 검증은 분리해야 한다. `go_router`가 올바른 경로를 만들어도 Release entitlement가 빠지면 Flutter 코드까지 도달하지 않는다. `curl`, `codesign`, 실기기만으로 1차 원인을 좁힐 수 있으므로 라우터 패키지를 곧바로 교체할 필요는 없다.

공식 기준은 [Flutter iOS Universal Links 설정](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links), [Apple Associated Domains 문서](https://developer.apple.com/documentation/xcode/supporting-associated-domains), [Apple Universal Links 디버깅 기술 문서](https://developer.apple.com/documentation/technotes/tn3155-debugging-universal-links)에서 확인할 수 있다.
