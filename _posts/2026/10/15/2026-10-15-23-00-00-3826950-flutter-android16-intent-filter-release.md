---
layout: post
title: "Flutter Android 16 Intent 오류 - release 진입점 검증과 intent-filter 대응"
description: "Flutter Android 16 release에서 Intent로 넘긴 route나 엔진 플래그가 무시될 때, deep link intent-filter와 release 검증 조건을 분리해 수정하는 방법을 정리한다."
date: 2026-10-15
tags: [Flutter, Android, release, deep link, 플랫폼통합]
comments: true
share: true
---

![Flutter Android 16 release Intent와 intent-filter 점검](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

Android 16(API 36) 출시를 준비하면서 Android `Intent`로 Flutter 화면을 열거나 `--route`를 넘기는 앱이라면, release에서 그 값을 계속 받을 수 있다고 가정하면 안 된다. 외부 링크로 화면을 열 목적이면 URI 기반 deep link와 `intent-filter`를 선언하는 선택이 맞고, 테스트 도구가 엔진 플래그를 주입하는 구조라면 profile로 분리해야 한다. 단순히 앱 아이콘을 눌러 시작하는 앱에는 이 대응이 필요하지 않다.

## 증상은 비슷하지만 원인은 두 갈래다

Flutter 앱에서 아래 두 상황을 한꺼번에 “Android 16 Intent 오류”라고 부르기 쉽다.

| 상황 | 실제 확인 대상 | 대응 |
|---|---|---|
| release에서 `--route`나 엔진 설정이 반영되지 않음 | Flutter Android embedder의 release 제한 | 빌드 시 설정·코드로 옮기고 `--use-application-binary` 주입을 제거 |
| 링크를 눌러도 앱이 열리지 않음 | AndroidManifest의 `intent-filter`와 URI | `ACTION_VIEW`, `DEFAULT`, `BROWSABLE`, scheme·host를 맞춤 |
| Android 16에서 Intent가 차단됐다는 logcat 경고 | `enforceIntentFilter`를 켠 컴포넌트 | 전달 Intent의 action·category·data를 필터와 일치시킴 |

Flutter 공식 변경 문서 기준으로 release 빌드는 Intent extra에서 `route`, `dart_entrypoint`, `dart_entrypoint_args` 같은 엔진 설정을 읽는 방식이 제한된다. 그래서 debug에서 `adb shell am start ... --es route /settings`가 동작했다는 사실만으로 Play 배포본도 같은 흐름을 보장하지 않는다.

## 외부 링크는 route extra 대신 URI로 표현한다

앱 밖에서 특정 화면을 열어야 한다면 아래처럼 AndroidManifest에 링크 계약을 적는다. 이 코드는 `/product/42` 같은 경로를 Flutter 라우터가 해석하게 만드는 최소 조건이다.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity
    android:name=".MainActivity"
    android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="example.com"
            android:pathPrefix="/product" />
    </intent-filter>
</activity>
```

앱 링크라면 `android:autoVerify="true"`와 함께 `https://example.com/.well-known/assetlinks.json`도 필요하다. 이 글의 핵심은 `route` 문자열을 Intent extra로 몰래 전달하는 것이 아니라, OS가 확인할 수 있는 action·scheme·host 계약으로 바꾸는 데 있다. Flutter의 `Router`나 `go_router`는 들어온 URI를 앱 상태로 변환하는 역할만 맡긴다.

검증할 때는 앱을 직접 실행하는 명령과 실제 링크 검증 명령을 구분한다.

```bash
# 필터에 맞는 앱이 설치되어 있는지만 확인한다.
adb shell am start -a android.intent.action.VIEW \
  -c android.intent.category.BROWSABLE \
  -d "https://example.com/product/42"

# Android가 도메인을 다시 검증하게 한다.
adb shell pm verify-app-links --re-verify com.example.app

# Android 16에서 Intent 차단 로그를 좁혀 본다.
adb logcat -s PackageManager
```

`am start`가 앱을 열었다고 해서 `assetlinks.json` 검증까지 끝난 것은 아니다. Play App Signing을 쓴다면 로컬 debug keystore가 아니라 Play Console의 앱 서명 SHA-256을 파일에 넣어야 한다. `http → https`, `example.com → www.example.com` 같은 서버 리다이렉트도 검증 실패 원인이 된다.

## `enforceIntentFilter`는 선택 기능이지만 미리 맞춰 둔다

Android 16의 Safer Intents는 현재 명시적으로 켠 컴포넌트에 적용된다. 앱 전체에서 켜려면 다음처럼 선언할 수 있다.

```xml
<application
    android:name="${applicationName}"
    android:label="example"
    android:icon="@mipmap/ic_launcher"
    android:intentMatchingFlags="enforceIntentFilter">
    ...
</application>
```

이 상태에서는 명시적으로 특정 컴포넌트를 지정한 Intent라도 그 컴포넌트의 필터와 맞아야 한다. `action` 없는 Intent를 필터와 연결하려는 코드, `BROWSABLE` 없이 브라우저 링크를 받으려는 코드, 등록하지 않은 scheme을 전달하는 코드를 먼저 정리해야 한다. Android 16 공식 문서는 차단 시 `Intent does not match component's intent filter` 또는 `Access blocked` 형태의 `PackageManager` 로그를 남긴다고 설명한다.

다만 이 옵션을 추가한다고 deep link가 자동으로 구현되는 것은 아니다. 필터가 너무 넓으면 예상하지 않은 외부 호출을 받을 수 있고, 너무 좁으면 정상 링크가 브라우저로 남는다. 실제 release APK에 병합된 Manifest를 확인한 뒤 URI별로 테스트해야 한다.

## 출시 전 체크리스트

- [ ] release에서 `--route`나 `--dart-entrypoint`를 Intent extra로 주입하지 않는다.
- [ ] 외부 화면 진입은 action·scheme·host·path를 가진 URI로 정의했다.
- [ ] `assetlinks.json`의 패키지명과 Play App Signing SHA-256이 일치한다.
- [ ] `http` 또는 `www` 리다이렉트 없이 `/.well-known/assetlinks.json`에 접근된다.
- [ ] `adb logcat -s PackageManager`에서 Intent 차단 경고가 없다.
- [ ] debug가 아니라 서명된 release APK/AAB 설치본에서 브라우저 링크를 눌러 확인했다.

Android 16에서 Intent 문제가 생겼을 때 모든 것을 Flutter 라우팅 버그로 보면 시간이 길어진다. release 엔진 설정 주입 문제인지, OS가 판단하는 deep link 계약 문제인지 먼저 나누면 된다. 테스트용 route 주입은 profile에 남기고, 실제 출시 경로는 Manifest와 URI 검증으로 고정하는 편이 안전하다.

참고 문서: [Flutter Android release 엔진 플래그 제한](https://docs.flutter.dev/release/breaking-changes/restrict-android-engine-flags-release-mode), [Flutter stricter Android entrypoint 검증](https://docs.flutter.dev/release/breaking-changes/stricter-android-entrypoint-intent-verification), [Flutter Android App Links](https://docs.flutter.dev/cookbook/navigation/set-up-app-links), [Android 16 동작 변경](https://developer.android.com/about/versions/16/behavior-changes-16)
