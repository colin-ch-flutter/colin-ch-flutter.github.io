---
layout: post
title: "Flutter Firebase App Check 출시 오류 - debug는 되는데 release에서 요청이 거절될 때"
description: "Flutter Firebase App Check가 debug에서는 동작하지만 release·Play 설치 앱에서 거절되는 원인을 provider, 서명, enforcement 순서로 점검한다."
date: 2026-10-02
tags: [Flutter, Firebase, Android, iOS, CI/CD]
comments: true
share: true
---

![Flutter Firebase App Check release 검증과 앱 보안](https://images.unsplash.com/photo-1563986768609-322da13575f3?auto=format&fit=crop&w=1600&q=80)

Flutter 앱에서 Firebase App Check를 붙인 뒤 debug에서는 Firestore나 Storage가 잘 되는데 release에서 요청이 거절된다면, 코드를 다시 고치기보다 **빌드 환경에 맞는 provider와 앱 등록 상태**를 먼저 확인하는 편이 빠르다. 로컬 에뮬레이터와 CI는 debug provider가 맞지만, 실제 Android 배포판은 Play Integrity, Apple 배포판은 DeviceCheck 또는 App Attest를 써야 한다. Firebase App Check를 사용하지 않는 앱이거나 enforcement를 켜지 않은 프로젝트라면 이 글의 복구 절차가 필요하지 않다.

## 오류가 release에서만 생기는 이유

App Check는 Firebase Authentication의 로그인 여부와 다른 계층이다. 로그인한 사용자라도 유효한 App Check 토큰이 없으면 enforcement가 켜진 Firestore, Realtime Database, Storage 등의 요청이 거절될 수 있다. debug provider로 발급받은 토큰을 release 빌드의 정상적인 인증으로 생각하면 문제가 생긴다.

| 실행 환경 | 권장 provider | 확인할 값 | 흔한 오판 |
| --- | --- | --- | --- |
| 에뮬레이터·로컬 개발 | Debug | Firebase Console에 등록한 debug token | release 검증으로 착각 |
| Android Play 설치 | Play Integrity | Play Console 앱 등록·서명 | 로컬 upload key를 사용 |
| iOS TestFlight·App Store | DeviceCheck 또는 App Attest | Bundle ID와 Apple capability | 시뮬레이터 성공을 실기기 성공으로 판단 |
| CI | Debug provider 또는 별도 테스트 전략 | CI용 token 등록 | production provider를 무조건 사용 |

## provider를 빌드 모드에 맞춰 나누기

Firebase 초기화 뒤, 다른 Firebase 서비스 호출 전에 App Check를 활성화해야 한다. debug token을 release에 넣지 않도록 빌드 모드로 분기한다.

```dart
import 'package:flutter/foundation.dart';
import 'package:firebase_app_check/firebase_app_check.dart';

Future<void> activateAppCheck() {
  return FirebaseAppCheck.instance.activate(
    androidProvider: kReleaseMode
        ? AndroidProvider.playIntegrity
        : AndroidProvider.debug,
    appleProvider: kReleaseMode
        ? AppleProvider.appAttestWithDeviceCheckFallback
        : AppleProvider.debug,
  );
}
```

`main()`에서는 `Firebase.initializeApp()`을 기다린 뒤 `activateAppCheck()`을 호출한다. 프로젝트에서 iOS 14 미만을 지원하거나 App Attest 조건을 충족하지 못하면 Apple provider를 DeviceCheck로 낮추고, 그 선택을 운영 문서에 남겨야 한다. 플러그인 버전에 따라 enum 이름이 달라질 수 있으므로 자동 완성에 나타나는 현재 `firebase_app_check` API를 확인한다.

Android release에서 특히 자주 빠지는 값은 SHA-256이다. 로컬 release APK의 upload key와 Google Play가 사용자에게 배포하는 APK의 Play App Signing key는 다를 수 있다. Play Console에서 앱의 App signing certificate SHA-256을 확인해 Firebase Android 앱 등록에 반영하고, 직접 설치 APK와 Play 설치 APK를 구분해 테스트한다.

## enforcement는 업데이트 뒤에 켠다

아직 App Check가 없는 기존 사용자가 있다면 다음 순서를 지키는 편이 안전하다.

1. Firebase Console에서 Android·iOS 앱을 각각 등록한다.
2. `firebase_app_check`를 추가하고 release 빌드를 TestFlight 또는 Play 내부 테스트 트랙에 배포한다.
3. Firestore·Storage·Functions 등 실제 사용하는 서비스의 App Check 지표에서 valid 요청 비율과 오류를 확인한다.
4. 문제가 없는 서비스 하나부터 enforcement를 켜고, 기존 버전 사용자 영향과 재설치·업데이트 경로를 확인한다.

App Check 토큰 TTL은 30분에서 7일 범위로 설정할 수 있다. 짧게 잡으면 재검증 빈도와 지연·quota 사용량이 늘고, 길게 잡으면 유출 토큰의 유효 시간이 길어진다. 특별한 근거 없이 가장 짧은 값으로 바꾸는 것은 해결책이 아니다.

## 출시 전 복구 체크리스트

- [ ] debug·profile·release가 서로 다른 provider를 사용한다.
- [ ] Android Play 설치 테스트에 Play App Signing SHA-256을 사용했다.
- [ ] iOS는 시뮬레이터가 아니라 실제 TestFlight 기기에서 확인했다.
- [ ] `Firebase.initializeApp()` 뒤, 첫 Firestore·Storage 호출 전에 App Check를 활성화했다.
- [ ] enforcement를 켜기 전 서비스별 지표를 확인했다.
- [ ] 로그에서 로그인 실패와 App Check 토큰 거절을 별도 오류로 분류했다.

Firebase 공식 기준으로 App Check는 모든 악용을 막는 장치가 아니며, 인증과도 대체 관계가 아니다. 이 문제에서 무료로 먼저 할 수 있는 일은 provider·서명·등록·enforcement 순서를 점검하는 것이다. 원인이 단순한 debug token 누락이라면 별도 보안 상품을 구매하거나 백엔드를 다시 만들 필요가 없다.

기준일은 2026년 10월 2일이다. Firebase 문서의 Flutter 기본 provider와 enforcement 절차, Android Play Integrity 조건을 기준으로 정리했으며, 실제 quota·지원 플랫폼은 배포 시점의 공식 문서를 다시 확인해야 한다.

참고 문서: [Firebase App Check Flutter 기본 provider](https://firebase.google.com/docs/app-check/flutter/default-providers), [Firebase App Check 개요와 quota](https://firebase.google.com/docs/app-check), [Firebase App Check debug provider](https://firebase.google.com/docs/app-check/flutter/debug-provider), [Flutter Android 출시와 서명](https://docs.flutter.dev/deployment/android)
