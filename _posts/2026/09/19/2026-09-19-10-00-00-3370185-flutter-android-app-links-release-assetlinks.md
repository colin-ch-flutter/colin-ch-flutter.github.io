---
layout: post
title: "Flutter Android App Links 출시 오류 - debug는 되는데 Play 설치에서 웹으로 열리는 이유"
description: "Flutter Android App Links가 debug 빌드에서는 열리지만 Play 설치 앱에서는 브라우저로 가는 문제를 assetlinks.json, Play App Signing SHA-256, adb 검증 순서로 복구한다."
date: 2026-09-19
tags: [Flutter, Android, 배포, CI/CD, 딥링크]
comments: true
share: true
---

![Flutter Android App Links와 assetlinks.json 출시 검증 흐름](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter Android App Links가 debug에서는 앱을 열지만 Play에서 설치한 release 앱에서는 브라우저로 간다면, `intent-filter`보다 서명 인증서와 `assetlinks.json`을 먼저 확인하는 선택이 유리하다. 아직 웹 도메인이 없거나 일반 커스텀 스킴이면 이 설정은 필요 없다. App Links는 `http`·`https` 도메인을 소유하고, 그 도메인과 Android 앱의 관계를 OS가 검증할 때만 동작한다.

## debug에서는 되고 Play 설치에서 실패하는 이유

로컬 debug APK는 debug 키로 서명된다. 반면 Play에서 내려받은 APK는 Play App Signing 키로 다시 서명된다. 따라서 서버 파일에 debug 인증서의 SHA-256만 넣어 두면 로컬 테스트는 통과해도 실제 사용자는 링크를 브라우저에서 보게 된다. Flutter 공식 문서도 Play App Signing을 사용하는 경우 Play Console의 `Release > Setup > App integrity > App signing`에 있는 지문을 사용하라고 안내한다.

| 확인 대상 | debug 설치 | Play 설치 | `assetlinks.json`에 넣을 값 |
| --- | --- | --- | --- |
| 서명 주체 | 로컬 debug 키 | Google Play App Signing 키 | 실제 배포 대상의 SHA-256 |
| 검증 목적 | 앱이 URL을 받을 수 있는지 | 도메인이 이 앱을 신뢰하는지 | package name과 인증서 일치 |
| 흔한 오판 | `adb`로 앱이 열림 | 링크를 눌러 브라우저가 열림 | debug 지문을 release에도 사용 |

## 출시용 설정은 두 곳이 한 쌍이다

AndroidManifest에는 도메인을 선언하고, 웹 서버에는 같은 앱의 인증서 지문을 게시한다. 아래 코드는 앱의 `MainActivity`에 넣는 최소 예시다.

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="example.com" />
</intent-filter>
```

이제 `https://example.com/.well-known/assetlinks.json`이 로그인 없이 200 응답을 주도록 배포한다. `package_name`은 `applicationId`와 같아야 하고, Play 설치를 검증한다면 Play Console의 App signing certificate SHA-256을 사용한다.

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.app",
      "sha256_cert_fingerprints": [
        "AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99"
      ]
    }
  }
]
```

debug와 내부 release APK도 함께 테스트해야 한다면 지문을 배열에 추가할 수 있다. 다만 flavor마다 `applicationId`가 다르면 지문만 추가하는 것으로 해결되지 않으며, 앱별 statement를 별도로 둬야 한다.

## 검증 순서는 앱 강제 실행과 구분한다

`adb shell am start`로 앱이 열렸다는 결과만으로는 도메인 인증 성공을 증명할 수 없다. Flutter 문서도 이 명령은 웹 파일 호스팅까지 검사하지 않는다고 명시한다. 설치된 실제 패키지에서 OS 검증 상태를 확인하는 순서는 다음과 같다.

```bash
# 설치된 앱의 도메인 승인 상태 확인
adb shell pm get-app-links com.example.app

# Android 12 이상에서 재검증 요청
adb shell pm verify-app-links --re-verify com.example.app

# 실제 브라우저 경로를 거치는 링크 실행
adb shell am start -a android.intent.action.VIEW \
  -c android.intent.category.BROWSABLE \
  -d "https://example.com/details/42"
```

먼저 브라우저에서 파일 URL을 열어 JSON 문법과 응답을 확인하고, 설치한 앱의 서명이 Play 인증서인지 확인한다. 테스트 기기에서 링크 기본 처리가 이미 꼬였다면 앱을 삭제 후 재설치한 뒤 인터넷 연결 상태에서 다시 검사한다. Android 공식 문서에 따르면 Android 15 이상은 `assetlinks.json` 변경이 기기에 반영되기까지 최대 7일이 걸릴 수 있어, 서버 파일을 고친 직후 사용자 기기 전체가 즉시 바뀐다고 가정하면 안 된다.

## 출시 전 체크리스트

- [ ] `android:host`와 실제 링크의 host가 완전히 같다.
- [ ] `assetlinks.json` 경로가 `.well-known` 아래이고 리다이렉트·인증 없이 접근된다.
- [ ] `package_name`이 debug manifest가 아니라 출시 `applicationId`와 같다.
- [ ] Play App Signing certificate SHA-256을 복사했다.
- [ ] `adb` 강제 실행과 브라우저에서 링크 클릭을 각각 확인했다.
- [ ] 로그인 전·후 목적지와 존재하지 않는 경로의 fallback을 확인했다.

핵심은 Flutter 라우터가 URL을 파싱하는 단계와 Android가 앱을 선택하는 단계를 나누는 것이다. 앱 내부 딥링크 테스트가 통과해도 `assetlinks.json`이나 release 서명이 틀리면 OS 단계에서 브라우저로 빠진다. 이 글의 기준일은 2026년 9월 19일이며, Android 15의 재검증 지연 조건은 배포 환경과 Google 서비스 탑재 여부에 따라 달라질 수 있다.

공식 확인 자료: [Flutter App Links 설정](https://docs.flutter.dev/cookbook/navigation/set-up-app-links), [Android App Links 검증](https://developer.android.com/training/app-links/verify-applinks), [Flutter Android 출시와 서명](https://docs.flutter.dev/deployment/android)
