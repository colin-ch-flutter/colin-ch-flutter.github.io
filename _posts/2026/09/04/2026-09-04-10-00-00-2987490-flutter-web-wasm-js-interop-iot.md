---
layout: post
title: "Flutter Web WASM과 dart:js_interop - IoT 대시보드 브라우저 연동 준비"
description: "Flutter Web WASM 빌드에서 dart:html과 package:js 오류를 피하고, dart:js_interop·package:web로 IoT 대시보드를 브라우저와 연동하는 방법을 정리했다."
date: 2026-09-04
tags: [Flutter, Dart, IoT, 스마트홈, 성능최적화]
comments: true
share: true
---

![Flutter Web WASM과 IoT 대시보드 브라우저 연동](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&q=80)

Flutter Web WASM은 IoT 대시보드를 브라우저에서 더 빠르게 실행할 수 있는 선택지다. 다만 기존 `dart:html`이나 `package:js`를 그대로 둔 채 `--wasm`만 붙이면 빌드가 깨진다. 이번에는 MQTT 화면은 유지하면서 브라우저의 알림·설정 페이지를 호출하는 코드를 `dart:js_interop`과 `package:web` 기준으로 바꿨다.

## `--wasm`만 붙이면 되는 줄 알았는데 아니었다

처음에는 아래 명령 하나면 끝날 거라고 생각했다.

```bash
flutter build web --wasm
```

실제로는 오래된 플러그인 안쪽에서 `dart:html`을 import하고 있어서 컴파일 단계에서 멈췄다. 모바일에서만 쓰던 패키지라도 Flutter Web 타깃에 포함되면 WASM 호환성을 검사받는다. 특히 BLE나 `dart:io`를 직접 사용하는 서비스는 브라우저 빌드 의존성에서 분리해야 한다.

| 코드 | WASM 기준 | 처리 |
| --- | --- | --- |
| `dart:html` | 지원하지 않음 | `package:web`로 교체 |
| `dart:js`, `package:js` | 지원하지 않음 | `dart:js_interop`으로 교체 |
| `dart:io`, 일부 네이티브 플러그인 | 브라우저에서 사용 불가 | 조건부 import 또는 Web 구현 |
| `dart.library.js` 조건 | WASM 구분에 부적합 | `dart.library.js_interop` 사용 |

## 브라우저 API 호출을 별도 어댑터로 격리한다

IoT 카드에서 브라우저 API를 직접 부르면 테스트와 모바일 빌드가 같이 복잡해진다. 아래처럼 웹 전용 어댑터를 만들고 화면에는 `openSettings()`만 노출하는 식으로 경계를 둔다.

```dart
// web_browser_adapter.dart
import 'dart:js_interop';
import 'package:web/web.dart' as web;

@JS('smartHomeOpenSettings')
external void _openSettings();

void openSettings() {
  // JS 함수가 없더라도 앱을 죽이지 않고 같은 탭에서 이동한다.
  try {
    _openSettings();
  } catch (_) {
    web.window.location.href = '/settings.html';
  }
}
```

`web/index.html`에는 앱이 호출할 함수를 등록한다. 실제 서비스에서는 이 함수 안에서 인증된 대시보드 URL을 조합하고, 사용자 입력값은 그대로 URL에 넣지 않는다.

```html
<script>
  globalThis.smartHomeOpenSettings = () => {
    globalThis.location.assign('/settings.html?from=flutter');
  };
</script>
```

모바일 구현까지 같은 파일에 넣으면 `package:web` 때문에 Android·iOS 컴파일이 실패한다. 이 경우 조건부 import로 구현을 나눈다.

```dart
// browser_adapter.dart
export 'browser_adapter_stub.dart'
  if (dart.library.js_interop) 'browser_adapter_web.dart';
```

이 방식의 장점은 MQTT Controller가 브라우저 타입을 알 필요가 없다는 점이다. `openSettings()`를 호출하고, 모바일에서는 네이티브 라우터나 아무 동작도 하지 않는 stub을 선택하면 된다.

## WASM을 켠 뒤 측정할 항목

WASM이라고 무조건 첫 화면이 빨라지는 것은 아니다. 우리 대시보드에서 먼저 비교할 값은 번들 다운로드 시간보다 메시지 수신 후 카드 렌더링 지연이었다.

| 확인 항목 | 측정 방법 | 판단 기준 |
| --- | --- | --- |
| 실제 실행 방식 | `bool.fromEnvironment('dart.tool.dart2wasm')` | WASM fallback 여부 확인 |
| 첫 화면 | Chrome Performance | 로딩·컴파일 구간 분리 |
| MQTT burst | 1초에 50개 메시지 주입 | 프레임 드랍과 입력 지연 확인 |
| 오류 추적 | `--source-maps` 빌드 | WASM 스택 심볼화 가능 여부 |

```bash
# QA에서는 스택을 읽기 쉽게, 배포 오류 추적에는 source map을 선택한다.
flutter build web --wasm --no-strip-wasm
flutter build web --wasm --source-maps
```

멀티스레드 렌더링까지 사용하려면 서버 응답 헤더도 필요하다. `Cross-Origin-Opener-Policy: same-origin`과 `Cross-Origin-Embedder-Policy: credentialless`(또는 `require-corp`)가 없으면 멀티스레드 경로가 활성화되지 않는다. 정적 호스팅에서 헤더 설정이 막혀 있다면 WASM 빌드가 되더라도 이 부분은 기대하지 않는 편이 맞다.

## 적용하면서 정리한 기준

Flutter Web WASM은 기존 앱을 버튼 하나로 바꾸는 최적화 옵션이 아니다. 웹 전용 API를 최신 interop으로 옮기고, 네이티브 의존성을 조건부 import로 떼어낸 뒤, 실제 브라우저에서 MQTT burst와 화면 입력을 측정해야 한다. `dart:html` 오류가 나면 앱 코드부터 의심하기보다 의존성 트리를 따라가면 원인을 빨리 찾을 수 있다.

공식 호환성 기준은 [Flutter WebAssembly 문서](https://docs.flutter.dev/platform-integration/web/wasm)와 [Flutter Web FAQ](https://docs.flutter.dev/platform-integration/web/faq)에서 확인할 수 있다.
