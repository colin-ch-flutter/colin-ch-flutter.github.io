---
layout: post
title: "Flutter iOS App Store 업로드 오류 - 암호화 수출 규정과 Info.plist 조건"
description: "Flutter iOS 앱을 TestFlight와 App Store에 올릴 때 암호화 수출 규정 질문이 반복되는 이유와 ITSAppUsesNonExemptEncryption 설정, 문서 제출이 필요한 조건을 정리한다."
date: 2026-10-13
tags: [Flutter, iOS, 배포·운영, CI/CD]
comments: true
share: true
---

![Flutter iOS App Store 암호화 수출 규정 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

그림에서 볼 부분은 앱 코드보다 App Store Connect 제출 정보와 빌드의 `Info.plist`가 출시 가능 여부를 좌우한다는 점이다.

Flutter iOS 앱을 TestFlight나 App Store에 올릴 때 암호화 질문이 반복된다면, HTTPS를 쓴다는 이유만으로 `YES`를 넣거나 모든 앱에 `NO`를 복사하면 안 된다. Apple 운영체제의 HTTPS처럼 면제되는 암호화만 사용한다면 `ITSAppUsesNonExemptEncryption`을 `NO`로 선언하는 경로가 유리하다. 자체 암호화 알고리즘이나 비면제 암호화 라이브러리를 포함했다면 `YES`와 수출 규정 문서 검토가 필요하다. 앱 기능과 연결된 암호화 종류를 확인할 수 없는 경우엔 이 글의 설정을 그대로 적용하면 안 된다.

Apple은 미국 외 지역에 배포하는 앱이 암호화를 사용하면 수출 규정 판단 대상이라고 설명한다. Dart 코드뿐 아니라 링크하는 서드파티 라이브러리도 확인해야 한다. `URLSession`의 HTTPS는 일반적으로 면제 대상이지만 앱 전체 구현에 따라 달라진다.

| 확인한 앱 조건 | `ITSAppUsesNonExemptEncryption` | 출시 단계 |
|---|---:|---|
| OS 제공 HTTPS 등 면제 암호화만 사용 | `NO` | 면제 근거를 기록해 둠 |
| 자체·비표준·비면제 암호화 사용 | `YES` | Apple 문서 제출과 심사 코드 확인 |
| 어떤 플러그인이 암호화를 포함하는지 모름 | 임의로 결정하지 않음 | 의존성 및 라이선스 확인 후 빌드 |

## Flutter 프로젝트에 선언하는 위치

반복 업로드에서 같은 질문을 줄이려면 Release 앱에 실제로 포함되는 `Info.plist`에 값을 둔다. `ios/Runner/Info.plist`에 아래 키를 추가하는 방식이다.

```xml
<!-- 앱과 포함된 라이브러리가 비면제 암호화를 쓰지 않는 경우에만 사용한다. -->
<key>ITSAppUsesNonExemptEncryption</key>
<false/>
```

이 값은 업로드를 통과시키는 마법의 플래그가 아니다. `flutter_secure_storage` 같은 플러그인 사용만으로 `YES`라고 단정할 수도, HTTPS만 쓴다고 검사 없이 `NO`라고 할 수도 없다. `pubspec.lock`, iOS Pod 의존성, 직접 작성한 암호화 코드를 확인한다.

비면제 암호화를 사용한다면 값을 `true`로 바꾸고 App Store Connect에 요구되는 문서를 제출한다. Apple 검토가 끝나면 제공되는 수출 규정 코드는 다음처럼 추가할 수 있다.

```xml
<key>ITSAppUsesNonExemptEncryption</key>
<true/>
<key>ITSEncryptionExportComplianceCode</key>
<string>APP_STORE_CONNECT에서_받은_코드</string>
```

## 업로드 전 체크리스트

- [ ] Release 빌드의 `ios/Runner/Info.plist`에 선언했는가
- [ ] Debug용 plist만 수정하고 Release 아카이브는 그대로 두지 않았는가
- [ ] `pubspec.lock`과 CocoaPods 의존성을 확인했는가
- [ ] `NO`를 선택했다면 OS 제공 또는 면제 암호화라는 근거를 기록했는가
- [ ] `YES`라면 문서 검토와 수출 규정 코드를 빌드·App Store Connect에 연결했는가
- [ ] `flutter build ipa`로 만든 아카이브의 plist를 다시 확인했는가

Apple 문서 기준으로 2026년 10월 13일 확인한 내용이다. 국가별 규정이나 패키지 구현이 바뀌면 다시 판단한다. plist와 의존성 점검은 무료로 할 수 있지만, 문서 제출 대상 앱을 `NO`로 숨기면 심사 재작업이 커질 수 있다.

핵심은 Flutter 문제가 아니라 배포 아카이브의 암호화 기능을 분류하는 일이다. 면제 조건을 확인한 앱만 `NO`를 선언하고, 비면제 암호화라면 문서 절차를 끝낸 뒤 TestFlight 빌드를 제출한다.

참고 문서: [Apple의 App Store Connect 수출 규정 개요](https://developer.apple.com/help/app-store-connect/manage-app-information/overview-of-export-compliance/), [Apple의 암호화 수출 규정 준수 안내](https://developer.apple.com/documentation/security/complying-with-encryption-export-regulations), [Flutter iOS 빌드·배포 안내](https://docs.flutter.dev/deployment/ios)
