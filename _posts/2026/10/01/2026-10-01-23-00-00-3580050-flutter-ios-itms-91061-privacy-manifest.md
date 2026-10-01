---
layout: post
title: "Flutter iOS ITMS-91061 오류 - 플러그인 PrivacyInfo.xcprivacy를 앱에 덧붙이면 안 되는 경우"
description: "Flutter iOS를 App Store에 올릴 때 ITMS-91061이 발생하면 SDK 매니페스트 누락인지 앱의 Required Reason API 선언 누락인지 구분하고, IPA에서 책임 번들을 확인해 재빌드하는 방법을 정리한다."
date: 2026-10-01
tags: [Flutter, iOS, CI/CD, 배포·운영]
comments: true
share: true
---
![Flutter iOS App Store ITMS-91061 Privacy Manifest 점검](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

Flutter iOS 앱 업로드 후 `ITMS-91061: Missing privacy manifest`가 나오면 먼저 메일의 경로가 앱인지 플러그인 프레임워크인지 나눈다. 별도 SDK라면 제공자의 매니페스트 포함 버전이 필요하고, 앱 실행 파일이 직접 Required Reason API를 호출했다면 앱 타깃을 고친다.

## 먼저 오류 번호와 책임 번들을 나눈다

| 경로 | 조치 |
|---|---|---|
| `Frameworks/Foo.framework/Foo` | Foo SDK 업데이트·교체. 앱 매니페스트에 복사하지 않는다 |
| 앱 실행 파일 | `Runner.app/PrivacyInfo.xcprivacy`에 실제 사용 이유를 선언한다 |
| 키·값 오류 | 문제 번들의 매니페스트를 Apple 허용 값으로 수정한다 |

확인 기준일은 2026년 10월 1일이다. 앱과 SDK는 각자 포함한 실행 파일의 사용 내역을 자신이 번들링한 매니페스트에 기록한다.

## IPA에서 실제 누락 위치를 확인한다

메일의 경로가 실제 아카이브에도 있는지 확인한다.

```bash
flutter build ipa --release

ARCHIVE="build/ios/archive/Runner.xcarchive"
APP="$(find "$ARCHIVE/Products/Applications" -maxdepth 1 -name '*.app' -print -quit)"

find "$APP" -name 'PrivacyInfo.xcprivacy' -print
find "$APP/Frameworks" -maxdepth 2 -type f -perm -111 -print
```

메일이 `Frameworks/Foo.framework/Foo`를 가리키는데 앱 매니페스트만 출력되면 SDK 문제는 남아 있다. 플러그인 버전과 `ios/Podfile.lock` 또는 Swift Package 변경을 확인하고, 그래도 없으면 교체하거나 공급자에게 수정 버전을 요청한다.

## 앱 코드의 선언만 직접 수정한다

앱이 자체 설정을 `UserDefaults`에 읽고 쓰는 경우처럼 실제 이유가 있을 때만 앱 매니페스트를 수정한다. `CA92.1`은 앱 자체 저장소일 때만 사용한다.

```xml
<key>NSPrivacyAccessedAPITypes</key>
<array>
  <dict>
    <key>NSPrivacyAccessedAPIType</key>
    <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
    <key>NSPrivacyAccessedAPITypeReasons</key>
    <array>
      <string>CA92.1</string>
    </array>
  </dict>
</array>
```

이 값을 넣어도 플러그인 누락은 해결되지 않는다.

## 출시 전 체크리스트

- [ ] 메일의 오류 번호와 누락 SDK 경로를 기록했다.
- [ ] `Runner.app`와 각 `*.framework`의 매니페스트 위치를 확인했다.
- [ ] 책임 번들을 수정한 동일 Release IPA를 다시 검사했다.

정리하면 `ITMS-91061`은 앱에 PrivacyInfo 파일 하나를 추가하라는 오류가 아니다. 메일 경로가 앱인지 플러그인 프레임워크인지 확인하고 책임 번들의 매니페스트를 업데이트해야 재업로드를 줄일 수 있다.

참고 문서: [Apple의 ITMS-91061 안내](https://developer.apple.com/forums/thread/774960), [Privacy manifest files](https://developer.apple.com/documentation/bundleresources/privacy-manifest-files), [Required Reason API 선언](https://developer.apple.com/documentation/bundleresources/describing-use-of-required-reason-api)
