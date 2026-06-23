---
layout: post
title: "Flutter로 IoT 앱 만들기 - 스마트홈 앱 개발 시작"
description: " "
date: 2026-04-30
tags: [Flutter, IoT, 스마트홈]
comments: true
share: true
---

# Flutter로 IoT 앱 만들기 - 스마트홈 앱 개발 시작

![스마트홈 IoT 기기](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## 시작하게 된 계기

솔직히 말하면, 처음부터 이 프로젝트를 하고 싶었던 건 아니었다. 사내에서 기존에 네이티브 Android/iOS로 따로 개발되던 앱을 Flutter로 통합해달라는 요청이 들어왔을 때, 처음 든 생각은 "아, IoT에 Flutter가 맞나?" 였다.

BLE(블루투스), MQTT, 각종 하드웨어 연동... 이런 걸 Flutter로 다 처리할 수 있을지 의구심이 있었다. 그런데 막상 시작해보니 생각보다 훨씬 Flutter 생태계가 잘 갖춰져 있었고, 결과적으로는 꽤 만족스러운 선택이었다고 생각한다.

## 스마트홈 앱이 뭐 하는 앱인가

한 줄로 요약하면 **스마트홈 가스IoT 기기를 스마트폰으로 원격 제어하는 앱**이다.

주요 기능을 나열해보면 이렇다:

- **기기 등록**: BLE(블루투스)나 QR 코드로 IoT 기기를 앱에 등록
- **원격 제어**: MQTT 프로토콜을 통해 온도 조절, 전원 On/Off, 운전 모드 전환
- **스케줄**: 요일별 / 시간별 온도 스케줄 설정 (출근 전에 미리 집 따뜻하게)
- **퀵 모드**: 자주 쓰는 설정을 프리셋으로 저장
- **자동화**: 특정 조건에서 자동으로 동작하도록 설정
- **멤버 관리**: 가족 구성원이 같은 기기를 함께 제어
- **IoT 모니터링 대시보드 IoT 기기**: 상업용 대형 IoT 기기 관리 기능
- **알림**: 기기 이상, 예열 완료 등 푸시 알림

그리고 글로벌 제품이라 다국어를 지원한다.

## 왜 Flutter를 선택했나

Flutter 이전에는 Android는 Kotlin, iOS는 Swift로 각각 개발했다. 코드가 두 벌이다 보니 수정이 생길 때마다 양쪽 다 반영해야 했고, 자연히 피처 개발 속도가 느렸다. 더 큰 문제는 Android 담당 개발자와 iOS 담당 개발자 간의 미묘한 UI 차이였다. 같은 기획인데 플랫폼마다 느낌이 달랐다.

Flutter를 선택한 이유를 정리해보면:

1. **단일 코드베이스**: 코드 한 번 작성하면 Android/iOS 모두 동작
2. **성숙한 생태계**: BLE, MQTT, Firebase 등 필요한 패키지들이 충분히 있었음
3. **빠른 UI 개발**: Hot reload 덕분에 UI 수정 → 확인 사이클이 빠름
4. **Dart 언어**: 익숙해지는 데 오래 걸리지 않음

물론 단점도 있다. 특히 BLE처럼 네이티브 플랫폼 의존성이 높은 기능은 Android와 iOS 동작이 미묘하게 달라서 플랫폼별로 따로 테스트하고 처리해야 하는 경우가 꽤 있었다. 이 부분은 나중에 BLE 시리즈에서 자세히 다룰 예정이다.

## 기술 스택 overview

앱의 전체 기술 스택을 정리해두면:

```
상태관리: GetX
통신 (클라우드): MQTT over WebSocket (AWS IoT Core)
통신 (기기 등록): BLE (BluFi 프로토콜)
로컬 DB: Realm
인증: JWT (Access + Refresh Token)
클라우드: Firebase (FCM, Crashlytics, Remote Config, Analytics)
다국어: easy_localization
차트: fl_chart
애니메이션: Lottie
QR 스캔: mobile_scanner
```

이 중에서 가장 까다로웠던 건 BLE+WiFi 프로비저닝과 AWS IoT Core MQTT 연결이었다. 특히 BluFi 프로토콜을 직접 구현해야 했는데, 이 부분이 나중에 다시 생각해도 진이 빠지는 작업이었다.

## 프로젝트 구조 미리보기

Clean Architecture 기반으로 구조를 잡았다. 크게 4개 레이어로 나뉜다:

```
lib/
├── core/          # 설정, 상수, 유틸리티
├── domain/        # 비즈니스 로직 (Repository 인터페이스, UseCase)
├── data/          # 데이터 구현체 (API, MQTT, BLE, DB)
└── presentation/  # UI (View, Controller, Binding)
```

처음에는 "이게 Flutter 프로젝트에 너무 과한 구조 아닌가" 싶었는데, 프로젝트가 커지면서 이 구조 덕분에 각 레이어를 독립적으로 수정할 수 있어서 꽤 도움이 됐다. 이 부분도 다음 편에서 자세히 다룬다.

## 앞으로의 연재 계획

이 시리즈는 총 50편으로 기획했다. 대략적인 흐름은:

1. **1-5일차**: 프로젝트 셋업 (아키텍처, GetX, 다국어, Firebase)
2. **6-12일차**: BLE 연결 (스캔, BluFi 프로토콜, QR, 오류 처리)
3. **13-18일차**: MQTT 통신 (AWS IoT Core, 토픽 설계, 실시간 제어)
4. **19-23일차**: 데이터 레이어 (Realm, Repository, JWT, API)
5. **24-30일차**: UI/UX (제어 화면, 차트, 다크모드, 스케줄)
6. **31-33일차**: 알림 (FCM, 로컬 알림)
7. **34-37일차**: 사용자 관리 (회원가입, 권한, 보안 저장소)
8. **38-42일차**: IoT 모니터링 대시보드 & FOTA
9. **43-47일차**: 고급 기능 (Remote Config, 타임존, 성능 최적화)
10. **48-50일차**: 배포 & 회고

순서대로 읽으면 Flutter로 IoT 앱을 처음부터 끝까지 만드는 흐름을 볼 수 있도록 구성했다.

다음 편은 Clean Architecture 적용기다. 왜 이 구조를 선택했고, 실제로 어떻게 구성했는지 적어볼 예정이다.
