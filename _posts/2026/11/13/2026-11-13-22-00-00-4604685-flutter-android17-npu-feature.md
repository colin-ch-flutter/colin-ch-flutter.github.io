---
layout: post
title: "Flutter Android 17 NPU 오류 - 온디바이스 AI가 막힐 때 FEATURE_NEURAL_PROCESSING_UNIT 선언"
description: "Flutter 온디바이스 AI 앱이 Android 17 release에서 NPU 접근에 실패할 때 필요한 manifest 선언과 CPU·GPU fallback 판단, 기기 검증 절차를 정리한다."
date: 2026-11-13
tags: [Flutter, Android, 배포, 성능최적화]
comments: true
share: true
---

![Flutter Android 17 온디바이스 AI와 NPU 접근 조건](https://images.unsplash.com/photo-1518770660439-4636190af475?auto=format&fit=crop&w=1600&q=80)

Android 17(API 37)을 target으로 올린 Flutter 온디바이스 AI 앱에서 NPU 가속이 사라진다면 `FEATURE_NEURAL_PROCESSING_UNIT` 선언부터 확인해야 한다. LiteRT NPU delegate, 제조사 AI SDK, deprecated NNAPI처럼 NPU에 직접 접근하는 앱은 manifest에 하드웨어 feature를 선언해야 한다. CPU나 GPU fallback만 사용하는 앱, 서버 API만 호출하는 앱에는 이 설정이 필요하지 않다.

## 직접 접근과 fallback 구분

Android 공식 기준은 AI 모델 사용 여부가 아니라 NPU 직접 접근 여부다.

| 앱 구성 | NPU 선언 | 출시 판단 |
|---|---:|---|
| 서버 API·CPU/GPU fallback | 불필요 | NPU 없는 기기에서도 기능 유지 |
| LiteRT NPU delegate·NNAPI 직접 호출 | 필요 | manifest와 기기 검증 |
| 제조사 NPU SDK | 필요할 수 있음 | 지원 SoC와 함께 확인 |

## Flutter Android manifest에 선언하기

NPU가 없는 기기까지 설치 대상에 포함하려면 `required="false"`로 선언하고, 실행 시 실제 feature 유무에 따라 delegate를 선택한다.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-feature
        android:name="android.hardware.npu"
        android:required="false" />

    <uses-permission android:name="android.permission.INTERNET" />
</manifest>
```

`required="true"`는 NPU 없는 기기를 제외한다. 일반 앱은 fallback을 남기고 `PackageManager.hasSystemFeature("android.hardware.npu")` 결과로 delegate를 선택한다.

## release 검증 순서

debug에서 빠른 것은 NPU 가속 성공이 아니라 CPU fallback일 수 있다. 아래 순서로 빌드와 런타임을 분리해서 본다.

1. `flutter build appbundle --release`로 출시 변형을 만든다.
2. `adb shell getprop ro.build.version.sdk`로 API 레벨을 확인한다.
3. 로그에 `NPU`, `GPU`, `CPU` 중 선택된 delegate를 남긴다.
4. NPU 실기기와 NPU 없는 에뮬레이터에서 실행한다.
5. NPU 실패 시 fallback이 동작하는지 확인한다.

에뮬레이터에서 모델이 돌아가도 NPU 검증이 끝난 것은 아니다. `required=true`라면 최소 기기 조건과 맞춰야 한다.

## 짧은 점검표

- [ ] target SDK 37 이상과 NPU 직접 접근 여부를 확인했다.
- [ ] 직접 접근한다면 `android.hardware.npu`를 선언했다.
- [ ] NPU 없는 기기에서 fallback을 실행했다.
- [ ] release 로그에 delegate와 추론 실패를 기록했다.

핵심은 Flutter 위젯 코드가 아니라 Android 17의 하드웨어 접근 계약이다. NPU가 필수가 아니면 `required=false`와 fallback을 유지하고, 전문 단말 앱일 때만 `required=true`와 지원 기기 목록을 함께 운영한다.

확인 기준일은 2026년 11월 13일이다. [Android 17 릴리스 노트](https://developer.android.com/about/versions/17/release-notes)와 [공식 변경 안내](https://developer.android.com/blog/posts/android-17-is-here)는 NPU feature 선언과 LiteRT·NNAPI 적용 조건을 설명한다. Flutter 모델의 플랫폼 조건은 [Google AI Edge Flutter 설정 문서](https://developers.google.com/edge/mediapipe/solutions/setup_flutter)에서 확인한다.
