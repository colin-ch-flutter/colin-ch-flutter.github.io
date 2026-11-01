---
layout: post
title: "Flutter Android 17 앱 서명 - PQC 하이브리드 키를 지금 바꿔야 하는가"
description: "Flutter Android 17의 APK Signature Scheme v3.2와 PQC 하이브리드 서명이 출시 키에 미치는 영향을 Play App Signing과 자체 키 관리 조건으로 나눠 정리한다."
date: 2026-11-01
tags: [Flutter, Android, 배포, CI/CD, 보안]
comments: true
share: true
---
![Flutter Android 17 PQC 하이브리드 APK 서명과 출시 검증을 표현한 이미지](https://images.unsplash.com/photo-1563013544-824ae1b704d3?auto=format&fit=crop&w=1600&q=80)

Flutter Android 앱을 Android 17(API 37)까지 출시하는 팀이라면, **Play App Signing을 쓰는 경우 지금 업로드 키를 바꿀 필요가 없다**. Google Play가 하이브리드 서명으로 업그레이드할 수 있는 선택을 제공할 때 전환하면 된다. 반대로 앱 서명 키를 직접 보관하는 조직은 Android 17의 v3.2 APK Signature Scheme을 검토해야 하지만, 기존 키를 덮어쓰는 방식으로 해결하면 업데이트가 끊길 수 있다. 단순히 `targetSdk`를 올린다고 PQC 서명이 자동으로 생기는 문제도 아니다.

이 그림에서 봐야 할 부분은 기존 클래식 키와 PQC 키가 하나의 검증 경로로 묶인다는 점이다.

## Android 17에서 달라진 서명 경계

Android 17은 기존 v3.0·v3.1 서명과 별도로 v3.2 하이브리드 블록을 도입했다. 이 블록은 RSA 또는 ECDSA 같은 클래식 서명과 ML-DSA 기반 PQC 서명을 함께 사용한다. Android 16 이하에서는 이 블록을 처리하지 않고 기존 서명 블록을 사용하므로, 낮은 버전과의 호환성 때문에 클래식 서명을 계속 유지해야 한다.

| 앱의 키 관리 방식 | 지금 할 일 | 피해야 할 판단 |
|---|---|---|
| Google Play App Signing | Play Console의 하이브리드 업그레이드 선택 대기 | 업로드 키를 앱 서명 키로 착각하기 |
| 자체 앱 서명 키 | 새 클래식 키와 PQC 키, 서명 계보를 함께 설계 | 기존 클래식 키를 재사용해 바로 혼합하기 |
| 로컬 debug APK만 테스트 | Android 17 기기에서 설치 경로만 확인 | debug 서명으로 출시 호환성을 판단하기 |

Flutter의 Dart 코드나 `pubspec.yaml`을 수정해서 해결하는 기능이 아니다. 실제 사용자가 받는 APK를 누가 서명하는지, 그리고 그 서명 계보가 유지되는지가 핵심이다. AAB를 Play에 올리는 구조라면 개발자가 만든 upload key와 Google Play가 최종 APK에 쓰는 app signing key를 분리해서 기록해야 한다.

## Play App Signing을 쓰는 Flutter 앱

Flutter 공식 Android 배포 문서와 Google Play 안내의 공통 경로는 Play App Signing을 사용하는 것이다. 이 경우 Google이 Android 17용 하이브리드 서명을 제공할 때까지 앱 빌드 설정을 억지로 바꿀 필요가 없다. `flutter build appbundle --release`로 만든 AAB의 업로드 서명과 기기에 전달되는 최종 APK의 앱 서명은 서로 다른 단계이기 때문이다.

출시 전에 아래 세 값만 문서에서 대조한다.

```text
package name       = Play Console 앱의 패키지명
upload key         = CI가 AAB 업로드에 사용하는 키
app signing key    = Google Play가 사용자 기기에 전달할 APK에 사용하는 키
```

Play Console에서 하이브리드 서명 전환 선택이 보이지 않는다면, Flutter 프로젝트에서 임의로 keystore를 교체하지 않는다. 키 변경은 새 앱을 만드는 작업이 아니라 기존 앱 업데이트의 신뢰 관계를 바꾸는 작업이라 복구 비용이 크다.

## 자체 서명 키를 관리하는 경우

자체 키를 쓰는 APK 배포나 별도 스토어가 대상이라면 Android 공식 v3.2 문서의 키 회전 절차를 따라야 한다. 핵심 조건은 세 가지다.

- 하이브리드 블록에는 클래식 서명자와 PQC 서명자가 모두 있어야 한다.
- 새 하이브리드 전환에는 새 클래식 키도 필요하다. 기존 클래식 키를 그대로 재사용하면 안 된다.
- 기존 설치본을 업데이트하려면 이전 서명 계보가 새 서명 계보에 연결돼야 한다.

도구 버전과 정확한 인자 조합은 배포 대상과 기존 계보에 따라 달라지므로, 오래된 `apksigner` 명령을 복사해 실행하는 방식은 위험하다. 새 키를 만든 뒤에는 최소한 서명 검증 결과와 Android 16 이하, Android 17 기기의 설치·업데이트를 각각 확인한다.

```bash
# 생성한 release APK의 인증서와 서명 스킴을 먼저 확인한다.
apksigner verify --verbose --print-certs app-release.apk

# Flutter 산출물 자체도 별도로 보존한다.
flutter build appbundle --release
```

첫 명령이 통과해도 “업데이트 가능”까지 증명하는 것은 아니다. 기존 Play 설치본 위에 업데이트되는지, 새 기기에 설치되는지, Android 16에서 열리는지를 분리해 확인해야 한다.

## 출시 체크리스트

- [ ] Play App Signing인지 자체 서명인지 구분했다.
- [ ] upload key와 app signing key의 역할을 CI 문서에 적었다.
- [ ] Play App Signing 사용자는 Google Play의 하이브리드 업그레이드 선택을 확인했다.
- [ ] 자체 키 사용자는 새 클래식 키·PQC 키·기존 서명 계보를 함께 백업했다.
- [ ] Android 16 이하에서 기존 앱 업데이트가 되는지 확인했다.
- [ ] Android 17에서 새 설치와 업데이트를 확인했다.
- [ ] AAB, keystore 메타데이터, 서명 검증 로그를 같은 릴리스 번호로 보관했다.

이번 변경은 Flutter SDK 버전만 올리면 끝나는 마이그레이션이 아니다. Play App Signing 앱은 Google의 전환 경로를 기다리는 선택이 안전하고, 자체 키 앱은 서명 계보를 설계한 뒤에만 v3.2 전환을 진행해야 한다. Android 17 대응의 출시 기준은 `targetSdk` 숫자가 아니라 **최종 APK의 서명 주체와 기존 설치본의 업데이트 가능성**이다.

참고 문서: [Flutter Android 앱 빌드·배포](https://docs.flutter.dev/deployment/android), [Android APK Signature Scheme v3.2](https://source.android.com/docs/security/features/apksigning/v3-2), [Android 17 기능 및 API](https://developer.android.com/about/versions/17/features), [Google Play App Signing](https://support.google.com/googleplay/android-developer/answer/9842756)
