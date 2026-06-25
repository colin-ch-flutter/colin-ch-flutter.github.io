---
layout: post
title: "Android & iOS 앱 스토어 배포 준비 - 아이콘부터 심사까지"
description: " "
date: 2026-06-17
tags: [Flutter, 앱배포, GooglePlay, AppStore, 앱심사]
comments: true
share: true
---
![앱 배포](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

## 앱 아이콘 생성

`flutter_launcher_icons` 패키지로 앱 아이콘을 자동으로 생성한다:

```yaml
# pubspec.yaml
flutter_icons:
  android: true
  ios: true
  image_path: "assets/icon/app_icon.png"
  remove_alpha_ios: true  # iOS는 투명도 없어야 함
  
dev_dependencies:
  flutter_launcher_icons: ^0.14.3
```

```bash
flutter pub run flutter_launcher_icons
```

이 명령 하나로 Android의 mipmap 폴더들과 iOS의 Assets.xcassets에 모든 해상도의 아이콘이 생성된다.

원본 이미지 크기는 1024x1024 픽셀, PNG 포맷이 좋다.

## Android 서명 설정

Release 빌드는 서명이 필요하다. Keystore 파일을 생성한다:

```bash
keytool -genkey -v \
  -keystore ~/upload-keystore.jks \
  -keyalg RSA -keysize 2048 \
  -validity 10000 \
  -alias upload
```

`android/key.properties` 파일 생성 (git에 올리지 않도록 주의):

```properties
storePassword=<password>
keyPassword=<password>
keyAlias=upload
storeFile=<path-to-keystore>/upload-keystore.jks
```

`android/app/build.gradle` 수정:

```groovy
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }
}
```

## iOS 인증서와 프로비저닝 프로파일

iOS 배포는 Apple Developer 계정이 필요하다:

1. **Distribution Certificate** 생성 (Xcode → Preferences → Accounts)
2. **App ID** 등록 (App Store Connect → Certificates, Identifiers & Profiles)
3. **Provisioning Profile** 생성 (App Store Distribution)
4. Xcode에서 자동 서명 설정:

```
Xcode → Runner Target → Signing & Capabilities
→ Automatically manage signing 체크
→ Team 선택
```

## 권한 사용 목적 문구 (iOS)

iOS 심사에서 권한 목적 문구가 불충분하면 거절된다:

```xml
<!-- ios/Runner/Info.plist -->
<key>NSBluetoothAlwaysUsageDescription</key>
<string>IoT 기기 기기 등록을 위해 블루투스 연결이 필요합니다. 블루투스를 통해 기기에 WiFi 정보를 전달합니다.</string>

<key>NSCameraUsageDescription</key>
<string>QR 코드를 스캔하여 기기를 등록하기 위해 카메라 접근이 필요합니다.</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>주변의 스마트홈 기기를 검색하기 위해 위치 접근이 필요합니다.</string>

<key>NSUserNotificationsUsageDescription</key>
<string>기기 이상 발생 및 스케줄 실행 시 알림을 받으시려면 알림 허용이 필요합니다.</string>
```

## 심사에서 거절당한 경험

첫 번째 심사 제출에서 거절을 한 번 받았다.

**거절 이유**: 블루투스 권한 목적 문구가 충분히 구체적이지 않다는 것이었다. "블루투스 기기 연결"이라고만 써두었는데, "어떤 기기에, 어떤 목적으로 연결하는지" 더 구체적으로 명시해달라는 요구였다.

수정 후 "IoT 기기 기기 등록 시 WiFi 설정 정보를 전달하기 위해 블루투스 연결이 필요합니다"로 변경하고 재제출하니 통과됐다.

## 앱 스크린샷

App Store Connect와 Google Play Console 모두 스크린샷이 필요하다. 실제 앱 화면을 캡처하거나, Sketch/Figma로 만든 목업을 사용한다.

iPhone용 스크린샷 요구 사항:
- 6.9인치 (iPhone 16 Pro Max)
- 6.5인치 (iPhone 14 Plus / 15 Plus)
- 5.5인치 (iPhone 8 Plus) - 선택사항이지만 권장

## 버전 관리

`pubspec.yaml`의 버전 형식: `version: 1.2.3+45`
- `1.2.3` → CFBundleShortVersionString (iOS) / versionName (Android) - 표시되는 버전
- `45` → CFBundleVersion (iOS) / versionCode (Android) - 빌드 번호 (매 배포마다 올려야 함)

---

드디어 마지막 편이다. 50일간의 여정을 돌아볼 시간이다.
