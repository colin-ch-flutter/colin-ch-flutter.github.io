---
layout: post
title: "Flutter Google Sign-In ApiException 10 - Play Store 설치본만 실패할 때"
description: "Flutter Google Sign-In이 로컬 release APK에서는 되지만 Google Play 설치본에서 ApiException 10으로 실패할 때, Play App Signing 인증서와 Firebase OAuth 설정을 점검하는 순서를 정리한다."
date: 2026-09-23
tags: [Flutter, Android, Firebase, GoogleSignIn, 배포]
comments: true
share: true
---

![Flutter Google Sign-In Play Store 출시 인증서 점검](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=1200&q=80)

로컬에서 만든 `release.apk`는 로그인되는데 Google Play에서 설치한 앱만 `PlatformException(sign_in_failed, ApiException: 10)`을 내면, Flutter 코드보다 서명 인증서부터 확인하는 편이 빠르다. 직접 설치한 APK는 내 upload keystore로 서명하지만 Play가 배포하는 APK는 Play App Signing 키로 다시 서명하기 때문이다. Play 출시 앱을 테스트하는 경우에만 해당하며, 사내 APK만 배포한다면 Play 인증서 등록은 필요하지 않다.

## 왜 APK와 Play 앱의 결과가 다른가

Android 출시에는 두 인증서가 있다. Flutter 공식 문서도 업로드 키와 최종 사용자에게 전달되는 app signing 키를 구분한다.

| 테스트 대상 | 실제 서명 주체 | Firebase에 필요한 지문 |
|---|---|---|
| `flutter run` | debug keystore | debug SHA-1, 필요하면 SHA-256 |
| 로컬 `flutter build apk --release` | upload/release keystore | release SHA-1, SHA-256 |
| Google Play 설치본 | Play App Signing key | Play Console의 SHA-1, SHA-256 |

따라서 release keystore 지문만 Firebase에 넣으면 로컬 APK는 성공하고 Play 버전은 실패할 수 있다. `ApiException: 10`은 대개 패키지명·인증서·OAuth 클라이언트 조합이 맞지 않는 `DEVELOPER_ERROR`다.

## Play 인증서를 먼저 등록한다

Play Console에서 `Test and release → App integrity → App signing`으로 이동해 **App signing key certificate**의 SHA-1과 SHA-256을 복사한다. `Upload key certificate`와 혼동하지 않아야 한다. Firebase 공식 FAQ도 Google Sign-In에는 release 지문과 Google Play Console 지문을 각각 등록하라고 안내한다.

Firebase Console의 `Project settings → Your apps → Android 앱 → SHA certificate fingerprints`에 두 값을 추가한다. 저장 뒤에는 새 `google-services.json`을 내려받아 `android/app/google-services.json`을 교체한다.

```bash
# 로컬 release/upload 키가 무엇인지 확인한다.
cd android
./gradlew signingReport

# Play Console에서 받은 인증서 파일도 직접 확인할 수 있다.
keytool -printcert -file deployment_cert.der | grep -E 'SHA1|SHA256'
```

위 명령의 값과 Play Console의 값이 다르면 정상일 수 있다. 전자는 업로드 키, 후자는 Play가 사용자에게 배포할 때 사용하는 키다. Play Console의 App signing key를 Firebase에 추가하는 것이 핵심이다.

## OAuth 설정과 앱 번들을 함께 확인한다

Firebase에서 `Authentication → Sign-in method → Google`이 활성화되어 있는지 확인한다. Google Cloud Console의 OAuth 2.0 클라이언트에도 Android 앱의 `package name`과 Play App Signing SHA-1 조합이 존재해야 한다. Firebase 문서가 안내하는 것처럼 서버용 Web client ID와 Android client ID를 서로 바꿔 넣지 않는다.

| 확인 지점 | 실패하기 쉬운 값 | 조치 |
|---|---|---|
| Android 앱 ID | `applicationId`와 Firebase 앱의 패키지명이 다름 | Gradle·Firebase 패키지명을 동일하게 맞춘다 |
| 인증서 | upload SHA만 등록 | Play App signing SHA-1과 SHA-256을 추가한다 |
| 설정 파일 | 예전 `google-services.json` | 지문 추가 후 다시 다운로드한다 |
| OAuth | Web client ID를 Android ID 자리에 사용 | SDK 문서의 용도에 맞는 ID를 사용한다 |

수정 뒤 `flutter clean`만 반복하는 것으로는 Play에 이미 올라간 AAB가 바뀌지 않는다. 새 `versionCode`로 AAB를 만들어 내부 테스트 트랙에 올리고, Play에서 내려받은 설치본으로 확인해야 한다.

```bash
flutter clean
flutter pub get
flutter build appbundle --release --build-number=42
```

Play Console의 내부 테스트는 실제 Play 배포 서명을 검증하는 경로다. 더 빠른 확인이 필요하면 App bundle explorer에서 생성된 기기별 APK를 내려받아 설치한다. 로컬 APK와 Play APK의 서명도 비교할 수 있다.

```bash
apksigner verify --print-certs app-release.apk
apksigner verify --print-certs downloaded-from-play.apk
```

## 출시 전 체크리스트

- [ ] debug, release/upload, Play App signing 지문을 따로 기록했다.
- [ ] Firebase Android 앱에 Play SHA-1과 SHA-256을 추가했다.
- [ ] 변경 후 `google-services.json`을 다시 받았다.
- [ ] Android `applicationId`와 Firebase 패키지명이 같다.
- [ ] Google Cloud OAuth 클라이언트에 Play 서명 조합이 있다.
- [ ] 로컬 APK가 아니라 내부 테스트의 Play 설치본에서 로그인했다.
- [ ] 실패가 계속되면 Google Play 서비스 계정 선택 화면 뒤의 logcat 원문을 저장했다.

핵심은 `flutter clean`이 아니라 **실제 사용자에게 전달된 APK의 서명 지문**이다. Play 설치본에서만 Google Sign-In이 실패한다면 upload 키를 다시 만드는 데 시간을 쓰지 말고, Play App Signing 인증서를 Firebase와 OAuth 설정에 등록했는지부터 확인하면 된다.

참고 문서: [Flutter Android 배포와 서명](https://docs.flutter.dev/deployment/android), [Firebase Android Google 로그인](https://firebase.google.com/docs/auth/android/google-signin), [Firebase Android 문제 해결 FAQ](https://firebase.google.com/docs/android/troubleshooting-faq), [Google Play App Signing](https://support.google.com/googleplay/android-developer/answer/9842756)
