---
layout: post
title: "Flutter Android 17 TLS 오류 - Certificate Transparency로 release 연결이 끊길 때"
description: "Flutter Android 17 앱에서 staging·사설 인증서 연결이 release에서 실패할 때 Certificate Transparency 적용 조건과 network security config 분기, 확인 절차를 정리한다."
date: 2026-11-06
tags: [Flutter, Android, 네트워크, 배포·운영, 성능최적화]
comments: true
share: true
---

![Flutter Android 17 release 앱의 TLS 인증서와 네트워크 설정 점검 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

Android 17(API 37)을 target으로 올린 Flutter 앱에서 release 빌드만 HTTPS 연결에 실패한다면, 서버가 사설 CA이거나 공개 인증서의 Certificate Transparency(CT) 로그 조건을 만족하는지부터 확인해야 한다. 인터넷의 공개 인증서를 쓰는 일반 앱은 설정을 추가할 필요가 없고, 사내 staging·장비 서버·자체 CA를 연결하는 앱만 아래 분기가 필요하다. Android 공식 문서 기준으로 Android 17 target 앱은 CT가 기본 활성화된다.

## 실패 조건을 먼저 나눈다

CT는 인증서가 공개 로그에 기록됐는지 검증하는 기능이다. Android 16은 기본 비활성화지만 Android 17(API 37)부터 target 앱에서 기본 활성화된다. 따라서 “debug에서는 되고 Play 출시 앱에서만 `SSLHandshakeException`이 난다”면 서버 교체보다 target SDK와 인증서 발급 경로를 같이 비교해야 한다.

| 연결 대상 | Android 17 target | 우선 판단 |
|---|---:|---|
| 공개 CA가 발급한 운영 도메인 | 정상 동작이 기본 | CT를 끄지 말고 인증서 체인·만료일을 확인한다 |
| 사내 CA 또는 self-signed staging | CT 로그 검증과 맞지 않을 수 있음 | custom trust anchor 또는 도메인별 CT 예외를 검토한다 |
| localhost·에뮬레이터 서버 | 별도 예외 구성이 적용됨 | debug 전용 CA와 release 설정을 분리한다 |

여기서 헷갈리기 쉬운 점은 `network_security_config.xml`이 모든 Flutter 네트워크 요청을 자동으로 고치는 만능 파일은 아니라는 것이다. Android 플랫폼 TLS를 사용하는 플러그인·네이티브 코드인지, Dart `HttpClient`나 Dio 경로인지 먼저 나눠야 한다.

## release 설정은 도메인 단위로 좁힌다

운영 도메인은 CT를 유지하고, 꼭 필요한 staging 도메인만 예외로 두려면 아래처럼 Android 리소스를 둔다. 이 코드는 공개 인증서를 쓰는 운영 서버까지 전역으로 우회하지 않도록 범위를 제한하는 목적이다.

`android/app/src/main/res/xml/network_security_config.xml`을 만들고 문제가 확인된 호스트에만 CT 예외를 선언한다.

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">staging.example.internal</domain>
        <certificateTransparency enabled="false" />
    </domain-config>
</network-security-config>
```

그 다음 Android 앱 매니페스트의 `<application>`에 파일을 연결한다.

```xml
<application
    android:name="${applicationName}"
    android:networkSecurityConfig="@xml/network_security_config"
    android:label="my_app">
</application>
```

사설 CA를 쓰는 경우에는 CT만 끄는 것으로 신뢰가 생기지 않는다. `res/raw/company_ca.pem`을 추가하고 `<trust-anchors>`에 연결하거나, 운영에서 검증 가능한 정식 공개 인증서로 교체해야 한다. 개발용 CA는 `debug-overrides`에만 넣어 release APK에 포함되지 않게 한다.

## Flutter에서 확인할 순서

1. `flutter build appbundle --release`로 실제 출시 산출물을 만든다.
2. Android 17 에뮬레이터에서 같은 API 호스트를 호출하고 `adb logcat`의 `SSLHandshakeException`, `CertPathValidatorException`을 확인한다.
3. debug와 release의 `targetSdk`, `networkSecurityConfig`, 인증서 체인을 비교한다.
4. 요청이 Android 플랫폼 HTTP 플러그인인지 Dart `HttpClient`인지 확인한다. 후자라면 Android XML만 바꾼 뒤 해결됐다고 판단하지 않는다.
5. 운영 공개 도메인에서 정상 연결되는지와 staging 예외가 의도한 호스트에만 적용되는지 회귀 확인한다.

CT 예외는 보안 기능을 끄는 설정이다. 임시 staging 서버를 살리려고 `<base-config>` 전체에 `enabled="false"`를 두거나 인증서 검증 자체를 우회하는 코드를 넣으면, release에서 중간자 공격 방어와 서버 신뢰 검증 범위를 함께 약화시킬 수 있다. 가장 좋은 해결은 staging에도 공개적으로 검증 가능한 인증서를 쓰는 것이며, 예외가 필요하면 도메인·빌드 변형·삭제 조건을 기록해 둬야 한다.

핵심은 Android 17에서 Flutter 코드만 고치는 것이 아니다. `targetSdk 37`, 인증서 발급 방식, 요청 경로, debug/release 리소스를 한 묶음으로 확인해야 한다. 공개 운영 서버만 쓰는 앱은 CT 설정을 추가하지 않아도 되지만, 사설 인증서를 쓰는 앱은 release APK 기준으로 TLS 실패를 재현하고 예외 범위를 최소화해야 한다.

참고 문서: [Android 네트워크 보안 구성](https://developer.android.com/privacy-and-security/security-config), [Android 17 동작 변경](https://developer.android.com/about/versions/17/behavior-changes-17), [Flutter Android 앱 빌드·배포](https://docs.flutter.dev/deployment/android) (2026-11-06 확인)
