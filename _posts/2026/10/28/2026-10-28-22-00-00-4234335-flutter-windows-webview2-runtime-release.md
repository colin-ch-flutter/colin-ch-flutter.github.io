---
layout: post
title: "Flutter Windows WebView2 오류 - MSIX 출시 PC에서 Runtime 누락을 판별하는 배포 조건"
description: "Flutter Windows WebView2가 개발 PC에서는 열리지만 고객 PC에서 초기화 실패할 때 Evergreen과 Fixed Runtime을 나누고, 출시 전 검사와 오류 안내를 구성하는 방법을 정리한다."
date: 2026-10-28
tags: [Flutter, Windows, MSIX, 배포, WebView2]
comments: true
share: true
---

![Flutter Windows 앱에서 WebView2 Runtime 배포 조건을 점검하는 화면](https://images.unsplash.com/photo-1488590528505-98d2b5aba04b?auto=format&fit=crop&w=1600&q=80)

Flutter Windows 앱에서 `WebView2 Runtime` 초기화 오류가 release PC에서만 난다면, 웹 페이지 코드보다 런타임 배포 방식을 먼저 확인하는 선택이 유리하다. 일반 소비자용 Windows 10·11 앱은 Evergreen Runtime을 전제로 안내하고, LTSC·Windows Server·인터넷이 차단된 현장 PC라면 Fixed Runtime을 설치 프로그램에 포함하는 편이 맞다. WebView가 필요 없는 앱에는 이 의존성을 추가할 이유가 없다.

## 개발 PC에서만 되는 이유

`webview_flutter_windows`와 `webview_windows`는 Flutter 엔진 안에 Chromium 전체를 넣지 않는다. 대상 PC에 Microsoft Edge WebView2 Runtime이 있어야 컨트롤러가 초기화된다. Windows 11이나 최신 Windows 10 개발 PC에서는 이미 설치돼 있어 문제를 놓치기 쉽지만, 깨끗한 Windows 10 설치본·Windows Server·LTSC에서는 빠질 수 있다.

| 배포 환경 | 기본 선택 | 놓치기 쉬운 조건 |
|---|---|---|
| 일반 Windows 10·11 | Evergreen | 사용자가 Runtime 설치를 막을 수 있음 |
| 사내망·키오스크·LTSC | Fixed Runtime | 설치 크기와 업데이트 책임이 늘어남 |
| WebView가 선택 기능 | 실행 시 검사 | 오류 화면 없이 기능을 숨기면 원인 파악이 어려움 |

Microsoft 문서 기준으로 Evergreen은 자동 업데이트되는 공유 런타임이고, Fixed는 앱이 특정 버전을 함께 배포하는 방식이다. Fixed Runtime은 약 100~250MB의 추가 용량이 생길 수 있으므로 “모든 MSIX에 넣기”가 정답은 아니다.

## 앱 시작 시 누락을 오류로 바꾸지 않기

아래처럼 WebView 초기화를 별도 단계로 감싸면 고객 PC에서 앱 전체가 종료되는 대신 설치 안내를 보여줄 수 있다. `initialize()`의 구체적인 예외 타입과 검사 API는 사용하는 플러그인 버전에 맞춰 확인해야 한다.

```dart
Future<bool> initializeWebView(WebviewController controller) async {
  try {
    await controller.initialize();
    return true;
  } catch (error, stackTrace) {
    // 실제 앱에서는 오류 코드와 Windows 버전을 함께 기록한다.
    debugPrint('WebView2 Runtime unavailable: $error\n$stackTrace');
    return false;
  }
}
```

초기화가 `false`를 반환하면 빈 WebView를 계속 재시도하지 말고, “WebView2 Runtime이 필요하다”는 안내와 공식 설치 경로를 제공한다. 네트워크가 막힌 기업 환경이라면 Evergreen 부트스트래퍼가 실패할 수 있으므로 설치 관리자에서 Fixed Runtime 또는 사내 배포 패키지를 선택해야 한다.

## 출시 전에 확인할 체크리스트

- [ ] WebView를 실제로 사용하는 기능과 사용하지 않는 기능을 분리했다.
- [ ] WebView2가 없는 깨끗한 Windows 10 환경에서 MSIX 설치를 확인했다.
- [ ] Windows Server·LTSC·오프라인 PC가 대상인지 제품 요구사항에 적었다.
- [ ] Evergreen이면 최초 실행 때 런타임 설치 실패 안내가 나온다.
- [ ] Fixed이면 런타임 버전 업데이트와 설치 크기를 릴리스 항목으로 관리한다.
- [ ] WebView 초기화 실패가 앱 전체 크래시로 이어지지 않는다.

여기서 `flutter build windows --release`가 성공했다는 사실은 WebView2 배포가 끝났다는 뜻이 아니다. 빌드는 개발 머신의 런타임을 사용했을 뿐이고, 고객 PC의 설치 조건까지 검증하지 않는다. MSIX 서명 오류와도 구분해야 한다. 설치는 되지만 WebView 화면만 실패한다면 인증서보다 Runtime 유무를 먼저 보는 순서가 짧다.

정리하면, 일반 PC 대상 앱은 Evergreen을 기본값으로 두고 누락 안내를 준비하면 된다. 업데이트를 통제해야 하는 LTSC·키오스크·망분리 환경은 Fixed Runtime의 용량과 보안 업데이트 책임까지 감당할 때만 선택한다. 기준은 개발 PC가 아니라 WebView2가 없는 실제 출시 환경이다.

공식 확인 자료: [Microsoft WebView2 배포 모델](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/distribution), [WebView2 런타임 가이드](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/webview2), [webview_flutter_windows 패키지 문서](https://pub.dev/packages/webview_flutter_windows), [Flutter Windows 배포 문서](https://docs.flutter.dev/deployment/windows)
