---
layout: post
title: "Flutter iOS TestFlight FCM 오류 - APNs 토큰이 null일 때 확인할 release 조건"
description: "Flutter iOS 앱이 TestFlight에서 Firebase Cloud Messaging을 받지 못하거나 getAPNSToken()이 null일 때, APNs 권한·entitlement·토큰 순서를 점검하는 방법을 정리한다."
date: 2026-10-06
tags: [Flutter, Firebase, iOS, 배포, 알림, CI/CD]
comments: true
share: true
---

![Flutter iOS TestFlight FCM과 APNs 토큰 흐름](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter iOS 앱을 TestFlight에서만 알림이 받지 못한다면 서버부터 고치기보다 **실제 기기에서 APNs 토큰을 받은 뒤 FCM API를 호출하도록 순서를 고치는 선택**이 우선이다. Xcode의 Push Notifications capability, `aps-environment` entitlement, Firebase에 등록한 APNs 키가 모두 맞아야 한다. iOS 시뮬레이터만 확인하는 경우에는 이 글의 release 검증이 적용되지 않는다.

## TestFlight에서만 실패하는 이유

FCM의 iOS 메시지는 APNs를 통해 전달된다. 따라서 Firebase 프로젝트에 앱을 추가한 것만으로는 충분하지 않다. Apple Developer의 App ID에 Push Notifications가 활성화되어야 하고, archive된 앱에도 `aps-environment`가 들어가야 한다. Firebase 공식 문서도 iOS SDK 10.4.0 이상에서 FCM API 호출 전에 APNs 토큰이 준비됐는지 확인하라고 안내한다.

| 확인 지점 | 정상 기준 | 실패할 때 보이는 현상 |
|---|---|---|
| Xcode Signing & Capabilities | Push Notifications 활성화 | TestFlight 토큰 등록 실패 |
| Runner.entitlements | Release archive에 `aps-environment=production` 포함 | APNs 등록 실패 |
| Firebase Cloud Messaging | 같은 Bundle ID용 APNs 키 등록 | FCM 토큰은 있지만 미수신 |
| Flutter 초기화 순서 | APNs 토큰 뒤 FCM API 호출 | `getToken()` 예외·null |

## 토큰을 기다린 뒤 FCM 토큰을 받는다

아래 코드는 iOS에서 APNs 토큰을 확인한 뒤 FCM 등록 토큰을 요청하는 최소 흐름이다. 앱 시작 때 한 번만 실행하지 말고 `onTokenRefresh`도 서버에 반영해야 한다.

```dart
Future<String?> registerIosMessaging(
  Future<void> Function(String token) savePushToken,
) async {
  final messaging = FirebaseMessaging.instance;

  await messaging.requestPermission(
    alert: true,
    badge: true,
    sound: true,
  );

  // iOS에서는 이 값이 준비되기 전 FCM API를 호출하지 않는다.
  final apnsToken = await messaging.getAPNSToken();
  if (apnsToken == null) {
    return null; // 다음 앱 시작 또는 재시도 시 다시 확인한다.
  }

  final fcmToken = await messaging.getToken();
  if (fcmToken != null) {
    await savePushToken(fcmToken);
  }

  messaging.onTokenRefresh.listen(savePushToken);
  return fcmToken;
}
```

`getAPNSToken()`이 null이면 무한 대기 대신 로그를 남기고 짧은 지연 후 재시도하거나 다음 앱 시작에서 다시 등록한다. 테스트는 iOS 실기기의 TestFlight 앱에서 권한을 허용한 뒤 APNs 토큰과 FCM 토큰을 따로 기록한다. 앱을 앱 전환기에서 강제 종료한 뒤에는 background 메시지를 위해 앱을 다시 열어야 한다.

## archive 결과를 확인하는 체크리스트

- [ ] TestFlight Bundle ID와 Firebase iOS 앱 Bundle ID가 같다.
- [ ] Release target의 Push Notifications와 archive entitlement를 확인했다.
- [ ] Firebase Console에 같은 Apple App ID용 APNs 키가 등록돼 있다.
- [ ] 실기기에서 APNs·FCM 토큰과 foreground·background 수신을 따로 확인했다.

APNs 토큰이 계속 null이면 Dart 코드보다 signing과 provisioning profile을 먼저 확인한다. 두 토큰이 모두 있는데 알림만 안 오면 payload, 권한 상태, Firebase 서버 응답을 분리해서 확인하고 환경별 토큰을 섞지 않는다.

핵심은 세 단계다. **Push capability와 production entitlement를 archive에 포함하고, APNs 토큰이 준비된 뒤 FCM 토큰을 요청하며, 토큰 갱신을 서버에 다시 저장한다.** 이 조건을 만족하면 TestFlight에서만 재현되는 `getAPNSToken() == null` 문제를 코드와 서명 설정 중 어디서 시작할지 좁힐 수 있다.

참고 문서: [Firebase Cloud Messaging Flutter 시작하기](https://firebase.google.com/docs/cloud-messaging/flutter/get-started), [Apple APNs 등록](https://developer.apple.com/documentation/usernotifications/registering-your-app-with-apns)
