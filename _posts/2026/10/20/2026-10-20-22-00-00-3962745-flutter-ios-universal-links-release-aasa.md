---
layout: post
title: "Flutter iOS Universal Links 오류 - 시뮬레이터는 되는데 release에서 Safari가 열릴 때"
description: "Flutter iOS Universal Links가 debug·시뮬레이터에서는 동작하지만 TestFlight와 실기기 release에서 Safari로 열릴 때 AASA, Associated Domains, CDN 캐시를 점검하는 방법을 정리한다."
date: 2026-10-20
tags: [Flutter, iOS, 배포·운영, 테스트, CI/CD]
comments: true
share: true
---

![Flutter iOS Universal Links와 AASA release 점검](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 부분은 앱 코드보다 도메인과 iOS 서명 설정이 Universal Links의 진입점을 결정한다는 점이다.

Flutter iOS 앱의 링크가 시뮬레이터에서는 앱을 열지만 TestFlight나 실기기 release에서는 Safari로 간다면, `go_router`부터 고칠 문제가 아닐 가능성이 크다. `Associated Domains` entitlement, `apple-app-site-association(AASA)`의 Team ID·Bundle ID, 그리고 Apple CDN 캐시를 순서대로 확인해야 한다. 커스텀 URL scheme만 필요한 앱이나 웹 도메인을 소유하지 않은 앱에는 이 방식이 적용되지 않는다.

## release에서만 Safari가 열리는 이유

Universal Links는 `https://` 링크를 앱으로 연결하는 iOS 기능이다. Flutter 공식 문서도 도메인의 `/.well-known/apple-app-site-association` 파일과 Xcode의 Associated Domains 설정을 함께 요구한다. 둘 중 하나만 맞으면 debug 실행 화면은 정상이어도 설치된 앱의 링크 연결은 실패한다.

특히 flavor를 쓰면 다음 값이 달라지기 쉽다.

| 점검 대상 | 자주 생기는 불일치 | 확인 기준 |
|---|---|---|
| AASA `appIDs` | production Bundle ID 대신 debug ID 입력 | `<Team ID>.<Bundle ID>` |
| entitlement | `applinks:example.com`만 있고 `www`가 없음 | 실제 링크의 host와 일치 |
| 경로 | `/details/*`만 허용했는데 `/product/1`을 테스트 | `components`의 `/` 패턴 확인 |
| 배포 상태 | 파일을 고쳤지만 기기에서 계속 이전 결과 | AASA CDN 캐시와 앱 재설치 확인 |

## AASA와 Xcode를 같은 값으로 맞추기

코드보다 앞서 production bundle ID에 맞는 AASA 파일을 배포한다. `.json` 확장자는 붙이지 않는다.

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAMID.com.example.app"],
        "components": [{"/": "/details/*"}]
      }
    ]
  }
}
```

이 파일이 실제로 외부에서 읽히는지 확인하려면 아래처럼 응답을 우선 확인한다. HTML 리다이렉트나 인증 페이지가 나오면 iOS가 읽을 AASA가 아니다.

```bash
curl -i https://example.com/.well-known/apple-app-site-association
```

Xcode에서는 Runner target의 Signing & Capabilities에 `Associated Domains`를 추가하고 `applinks:example.com`을 넣는다. `www.example.com`으로 링크를 만들었다면 `example.com`만 등록해서는 안 된다. production flavor의 entitlements 파일이 별도로 있다면 Debug 설정만 고친 것은 아닌지도 확인한다.

## 출시 전 재현 체크리스트

Flutter 공식 문서 기준으로 Simulator는 `xcrun simctl openurl`로 테스트할 수 있지만, 최종 판정은 실제 서명된 앱에서 해야 한다.

1. TestFlight 앱을 삭제하고 다시 설치한다.
2. Notes에 `https://example.com/details/1`을 입력해 링크를 누른다.
3. 앱이 열리지 않으면 Safari가 아니라 링크를 길게 눌러 앱 열기 선택지가 있는지 본다.
4. AASA를 수정했다면 즉시 재시험하지 말고 CDN 반영 시간을 고려한다.
5. `TEAMID`, production Bundle ID, host, path를 한 줄씩 대조한다.

Flutter 문서는 Apple CDN이 AASA를 요청하기까지 최대 24시간 걸릴 수 있다고 안내한다. 따라서 파일을 고친 직후 Safari가 계속 열리는 것만으로 JSON이 틀렸다고 단정하면 안 된다. 반대로 하루가 지나도 실패하면 캐시 탓으로 넘기지 말고 HTTP 응답 헤더, 리다이렉트, entitlement, 앱 재설치를 다시 확인해야 한다. Apple의 Universal Links 디버깅 문서도 AASA 내용과 HTTP 응답 헤더를 별도로 점검하도록 안내한다.

## 짧은 요점 정리

- release 전용 실패는 Flutter 라우터보다 entitlement·AASA 불일치를 우선 확인한다.
- AASA의 `appIDs`는 production `<Team ID>.<Bundle ID>`여야 한다.
- 도메인의 `www` 유무와 `components` 경로가 실제 링크와 같아야 한다.
- AASA 수정 직후의 실패는 Apple CDN 캐시일 수 있으므로 TestFlight 재설치와 시간을 함께 기록한다.

참고: [Flutter Universal Links 공식 문서](https://docs.flutter.dev/cookbook/navigation/set-up-universal-links), [Apple Universal Links 디버깅 문서](https://developer.apple.com/documentation/technotes/tn3155-debugging-universal-links)
