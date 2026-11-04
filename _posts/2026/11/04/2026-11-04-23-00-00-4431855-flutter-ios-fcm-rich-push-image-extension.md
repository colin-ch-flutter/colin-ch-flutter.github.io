---
layout: post
title: "Flutter iOS FCM 이미지 알림 오류 - release에서 Notification Service Extension이 동작하지 않을 때"
description: "Flutter iOS FCM 푸시 본문은 오지만 이미지가 빠지는 release 오류를 Notification Service Extension, Podfile, mutable-content, 실기기 검증 조건으로 나눠 해결한다."
date: 2026-11-04
tags: [Flutter, Firebase, iOS, 배포, 테스트]
comments: true
share: true
---

![Flutter iOS FCM 리치 푸시와 Notification Service Extension 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

그림에서 볼 부분은 Flutter 화면 코드가 아니라 iOS 앱과 별도 Notification Service Extension이 함께 출시돼야 이미지가 보인다는 점이다.

Flutter iOS에서 FCM 알림의 제목과 본문은 오는데 이미지가 빠진다면 `firebase_messaging` 코드보다 Notification Service Extension을 우선 확인하는 편이 맞다. 이미지가 필요 없는 일반 알림에는 이 구성이 필요 없다. 특히 iOS 시뮬레이터는 Firebase 공식 문서상 이미지 알림 검증 대상이 아니므로 실기기에서 확인해야 한다.

## 본문만 오는 이유

FCM의 rich notification은 알림이 표시되기 직전에 iOS 확장이 이미지 URL을 내려받아 첨부한다. 따라서 아래 세 단계 중 하나라도 빠지면 알림 자체는 오지만 이미지가 사라진다.

| 확인 지점 | 빠졌을 때 보이는 현상 | 수정 위치 |
|---|---|---|
| Notification Service Extension | 본문만 표시 | Xcode의 별도 Extension target |
| `Firebase/Messaging` pod | 확장 빌드 또는 실행 실패 | `ios/Podfile` |
| `mutable-content: 1` | 확장이 호출되지 않음 | FCM payload |

## Extension target과 Podfile 연결

Xcode에서 `Runner.xcworkspace`를 열고 File → New → Target → Notification Service Extension을 선택한다. 이름은 예시로 `ImageNotification`을 사용한다. `.xcodeproj`가 아니라 workspace를 열어야 CocoaPods 의존성이 같은 프로젝트에 연결된다.

Extension이 Firebase Messaging을 사용할 수 있도록 Podfile에 별도 target을 둔다.

```ruby
target 'ImageNotification' do
  pod 'Firebase/Messaging'
end
```

그 뒤 `ios` 폴더에서 `pod install`을 실행하고, Xcode의 Extension target에 Development Team과 배포 타깃을 설정한다. Runner와 Extension의 최소 iOS 버전이 다르면 archive 단계에서 실패할 수 있으므로 두 target을 같은 값으로 맞춘다.

## FCM payload 조건

이미지가 포함된 알림은 `aps.mutable-content`가 1이어야 한다. `notification.image`만 넣고 이 값을 생략하면 iOS가 확장을 실행하지 않는다. 이미지 URL은 실기기에서 인증 없이 접근 가능하고, 서버가 올바른 MIME 타입을 반환해야 한다.

```json
{
  "message": {
    "token": "DEVICE_TOKEN",
    "notification": {
      "title": "보일러 알림",
      "body": "거실 온도가 설정값보다 낮다",
      "image": "https://example.com/boiler.jpg"
    },
    "apns": {
      "payload": {
        "aps": {
          "mutable-content": 1
        }
      }
    }
  }
}
```

Flutter의 `FirebaseMessaging.onMessage`가 정상 호출되는 것과 백그라운드 rich notification이 동작하는 것은 다른 경로다. 전자는 Dart 앱 실행 여부를 확인하지만, 후자는 iOS가 Extension을 호출했는지와 이미지 다운로드가 성공했는지를 확인해야 한다.

## release에서 확인할 순서

```bash
cd ios
pod install
cd ..
flutter build ipa --release
```

archive가 만들어지면 Runner만 보지 말고 `.xcarchive/Products/Applications/Runner.app/PlugIns` 아래에 `ImageNotification.appex`가 들어갔는지 확인한다. 없다면 Extension embed 설정이나 Build Phases 순서를 의심한다. 앱을 TestFlight에 올린 뒤에는 실기기에서 앱이 백그라운드인 상태로 HTTPS 이미지 URL을 포함한 알림을 보낸다.

- [ ] Xcode에서 `Runner.xcworkspace`를 열었다.
- [ ] Extension target에 `Firebase/Messaging` pod가 연결됐다.
- [ ] Runner와 Extension의 iOS 배포 타깃이 일치한다.
- [ ] payload에 `mutable-content: 1`이 있다.
- [ ] 이미지 URL이 로그인 없이 200 응답과 이미지 MIME 타입을 반환한다.
- [ ] Debug가 아니라 TestFlight 또는 release 서명 실기기에서 확인했다.

이미지가 계속 빠져도 Flutter 위젯을 고치는 것으로 해결되지 않을 수 있다. Apple은 Notification Service Extension을 원격 알림을 표시하기 전에 내용을 수정하는 별도 객체로 정의하고, Firebase는 iOS 이미지 표시를 위해 별도 Extension과 `Firebase/Messaging` 연결을 요구한다. 반대로 단순 텍스트 알림만 필요하다면 Extension을 추가해 빌드 복잡도를 늘릴 이유가 없다.

참고 기준은 2026년 11월 4일 확인한 [Firebase Flutter FCM 수신 문서](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages), [Apple UNNotificationServiceExtension 문서](https://developer.apple.com/documentation/usernotifications/unnotificationserviceextension), [Flutter iOS 앱 확장 문서](https://docs.flutter.dev/platform-integration/ios/app-extensions)다. 실제 앱의 서버 payload와 서명 인증서는 프로젝트마다 다르므로 위 JSON의 토큰과 URL을 그대로 사용하면 안 된다.
