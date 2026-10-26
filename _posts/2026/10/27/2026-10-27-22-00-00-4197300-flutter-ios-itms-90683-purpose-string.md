---
layout: post
title: "Flutter iOS ITMS-90683 오류 - Release Info.plist의 권한 설명 누락 점검"
description: "Flutter iOS 앱을 App Store에 올릴 때 ITMS-90683이 발생하면 보호 리소스별 Usage Description과 플러그인 참조 여부, Release 산출물을 확인해 수정하는 방법을 정리한다."
date: 2026-10-27
tags: [Flutter, iOS, AppStore, 배포, CI/CD]
comments: true
share: true
---

![Flutter iOS App Store ITMS-90683 권한 설명 점검](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 것은 iOS 앱 기능보다 App Store 제출 산출물의 `Info.plist`가 권한 사용 이유를 설명하는지다.

Flutter iOS 앱 업로드 뒤 `ITMS-90683: Missing purpose string in Info.plist`가 나오면 `ios/Runner/Info.plist`에 아무 키나 추가하는 방식은 위험하다. 메일에 나온 보호 리소스와 실제 Release 앱의 `Info.plist`를 맞춰야 한다. 카메라·사진·마이크·위치 기능을 실제로 쓰는 앱은 설명 문구를 추가하고, 플러그인이 불필요한 API를 끌어왔다면 플러그인 업데이트나 제거를 검토해야 한다. 해당 기능이 전혀 없는 앱에 그럴듯한 권한 문구만 넣는 선택은 맞지 않다.

## 오류가 생기는 조건

Apple은 앱 코드뿐 아니라 외부 라이브러리와 SDK가 보호 리소스 API를 참조하는 경우에도 앱 개발자에게 설명 문자열을 요구한다. 그래서 Dart 코드에서 카메라를 호출하지 않아도 `image_picker`, 녹음, 위치, Bluetooth 계열 플러그인의 iOS 코드를 링크했다면 업로드 검사에 걸릴 수 있다. Apple 공식 문서가 예시로 드는 오류는 `NSLocationWhenInUseUsageDescription` 누락이며, 실제 메일의 키가 가장 정확한 단서다.

| 메일에 표시된 키 | 흔한 Flutter 기능 | 확인할 파일·조건 |
|---|---|---|
| `NSCameraUsageDescription` | 카메라 촬영, QR 스캔 | `image_picker`, QR 플러그인과 실제 촬영 기능 |
| `NSMicrophoneUsageDescription` | 녹음, 영상 통화 | 오디오·통화 플러그인과 녹음 여부 |
| `NSPhotoLibraryUsageDescription` | 사진 선택·읽기 | 사진을 읽는지, 추가만 하는지 구분 |
| `NSLocationWhenInUseUsageDescription` | 지도, 위치 기반 BLE | 포그라운드 위치 사용 여부 |

## Release 설정부터 고친다

아래처럼 앱이 실제로 요청하는 이유를 사용자 관점의 한 문장으로 작성한다. 제품 기능과 관계없는 문구를 복사하면 App Review에서 다시 설명해야 한다.

```xml
<!-- ios/Runner/Info.plist: 앱 기능에 해당하는 키만 추가한다. -->
<key>NSCameraUsageDescription</key>
<string>매장 QR 코드를 스캔해 주문 정보를 불러오기 위해 카메라를 사용한다.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>프로필 사진을 선택해 계정 화면에 표시하기 위해 사진 보관함을 사용한다.</string>
```

사진을 읽지 않고 저장만 한다면 `NSPhotoLibraryAddUsageDescription`이 더 좁은 선언일 수 있다. 위치도 백그라운드 추적이 아니라 화면을 보는 동안만 필요하다면 `NSLocationWhenInUseUsageDescription`과 실제 코드의 권한 요청 범위를 맞춘다. 문구를 넣었는데도 계속 실패하면 Debug 파일만 수정했거나, Xcode Build Settings가 `Info-Release.plist`를 바라보는 프로젝트일 수 있다. Flutter 공식 안내처럼 Debug와 Release에 다른 plist를 쓰는 구성이라면 Release 파일에도 같은 검토를 적용한다.

## IPA에서 누가 API를 참조했는지 확인한다

빌드 성공은 App Store 메타데이터 검사를 통과했다는 뜻이 아니다. Archive 또는 IPA를 만든 뒤 앱 번들의 최종 plist를 확인하고, 메일의 키가 실제 산출물에 있는지 검사한다.

```bash
flutter build ipa --release

# 산출물 경로는 프로젝트 설정에 따라 달라질 수 있다.
unzip -l build/ios/ipa/*.ipa | rg 'Payload/.app/Info.plist'
plutil -p build/ios/archive/Runner.xcarchive/Products/Applications/*.app/Info.plist \
  | rg 'UsageDescription|Bluetooth|Location|Camera|Microphone|Photo'
```

키가 없으면 Release 타깃의 plist 경로와 빌드 설정을 고친다. 키는 있는데도 ITMS-90683이 지속되면 메일에 표시된 프레임워크가 책임 번들일 수 있다. 이때 앱의 `Info.plist`에 임의의 문구를 더하기보다 `Pods/`, Swift Package, 플러그인 버전을 확인하고, 해당 공급자가 안내한 iOS 버전으로 올린 뒤 다시 Archive한다. 실제 플러그인이 그 리소스를 쓰지 않는다면 제거·대체가 더 정직한 해결이다.

## 출시 전 체크리스트

- [ ] App Store 메일의 누락 키를 그대로 기록했다.
- [ ] Release 타깃이 사용하는 plist에 해당 키가 들어 있다.
- [ ] 문구가 “앱 기능에 왜 필요한지”를 설명하며 빈 문자열이 아니다.
- [ ] 플러그인이 참조한 API라면 최신 버전의 매니페스트·권한 요구사항을 확인했다.
- [ ] IPA의 최종 `.app/Info.plist`를 열어 Archive 결과를 확인했다.
- [ ] 기능을 쓰지 않는 권한 키와 플러그인을 제거할 수 있는지 판단했다.

Apple 기준 확인일은 2026년 10월 27일이다. 목적 문자열은 4,000바이트보다 짧고 구체적이어야 하며, 외부 SDK가 접근하는 보호 리소스도 앱의 책임 범위에 포함된다. 따라서 `ITMS-90683`은 권한 문구 한 줄을 채우는 오류가 아니라, Release 바이너리에 포함된 API와 사용자에게 고지한 기능의 범위를 맞추는 출시 점검 항목이다.

참고 문서: [Apple protected resource와 ITMS-90683 안내](https://developer.apple.com/documentation/uikit/requesting-access-to-protected-resources), [Apple App Review 목적 문자열 가이드](https://developer.apple.com/help/app-review/guideline-reference/5-1-1-purpose-strings), [Flutter iOS Debug·Release Info.plist 구성](https://docs.flutter.dev/add-to-app/ios/project-setup)
