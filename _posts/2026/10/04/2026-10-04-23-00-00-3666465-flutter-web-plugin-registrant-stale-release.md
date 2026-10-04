---
layout: post
title: "Flutter Web release MissingPluginException - 플러그인 추가 뒤 web_plugin_registrant가 갱신되지 않을 때"
description: "Flutter Web release에서 새 플러그인이 등록되지 않아 MissingPluginException이나 channel-error가 날 때, stale web_plugin_registrant를 확인하고 CI 캐시를 안전하게 초기화하는 방법을 정리한다."
date: 2026-10-04
tags: [Flutter, 웹, 테스트, CI/CD, 성능최적화]
comments: true
share: true
---

![Flutter Web release 플러그인 등록 파일과 CI 캐시 점검](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

Flutter Web에서 플러그인을 추가한 뒤 release만 `MissingPluginException`이나 `channel-error`가 난다면, 플러그인 코드보다 오래된 `web_plugin_registrant.dart`를 먼저 의심해야 한다. CI 워크스페이스나 로컬 `build/`를 재사용하는 앱에 해당한다.

## 증상과 원인

Flutter Web 플러그인은 앱 빌드 과정에서 웹용 등록 파일을 생성한다. 기존 release 캐시를 재사용하면 `pubspec.yaml`에 새 플러그인을 추가해도 등록 파일에는 이전 플러그인만 남을 수 있다. 빌드는 성공하지만 새 기능을 호출하는 순간 오류가 난다.

| 증상 | 판단 |
| --- | --- |
| 빌드는 성공하고 새 플러그인 호출만 실패 | 등록 파일 누락 가능성 |
| `flutter clean` 뒤 정상 동작 | stale build 캐시 가능성 높음 |
| 새 checkout에서도 실패 | 플러그인 웹 지원·조건부 import 확인 |

## 복구 절차

플러그인 추가 후 release 결과물을 다시 만들 때는 생성 산출물과 `.dart_tool`을 함께 버리고 의존성을 다시 해석한다.

```bash
flutter clean
flutter pub get
flutter build web --release
```

등록 파일에 새 플러그인의 `registerWith`가 들어갔는지도 확인할 수 있다.

```bash
rg -n "registerWith|새_플러그인" .dart_tool/flutter_build -g 'web_plugin_registrant.dart'
```

CI에서는 `build/`와 `.dart_tool/`를 무조건 복원하지 않는다. Flutter SDK 버전과 `pubspec.lock`을 캐시 키에 포함하고, 의존성이 바뀌면 `flutter clean`을 실행한다.

## 계속 실패할 때

패키지가 실제로 Web 구현을 제공하는지 pub.dev의 플랫폼 목록과 패키지 `pubspec.yaml`의 `flutter.plugin.platforms.web`를 확인한다. 모바일 전용 플러그인은 등록 파일을 새로 만들어도 구현이 생기지 않는다. `dart:io` 같은 모바일 전용 코드는 조건부 import로 분리해야 한다.

- [ ] 플러그인 추가 뒤 `flutter pub get`을 실행했다.
- [ ] `flutter clean` 후 release를 다시 만들었다.
- [ ] 생성된 `web_plugin_registrant.dart`에 새 플러그인이 들어갔다.
- [ ] CI 캐시가 `build/`와 `.dart_tool/`를 오래된 키로 복원하지 않는다.
- [ ] 패키지가 Web 플랫폼을 지원한다.

Flutter Web release에서 새 플러그인만 작동하지 않을 때는 난독화나 브라우저 캐시보다 등록 파일과 `.dart_tool` 캐시를 먼저 비교한다. `flutter clean`으로 해결되면 CI 캐시 키를 의존성 변경과 연결해야 재발하지 않는다.

참고 문서: [Flutter Web 플러그인 개발](https://docs.flutter.dev/packages-and-plugins/developing-packages), [Flutter Web release 빌드](https://docs.flutter.dev/deployment/web), [stale web_plugin_registrant 재현 이슈](https://github.com/flutter/flutter/issues/189128)
