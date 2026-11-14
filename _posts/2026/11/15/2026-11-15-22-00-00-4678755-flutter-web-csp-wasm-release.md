---
layout: post
title: "Flutter Web CSP 오류 - release에서 flutter_bootstrap.js와 WASM이 차단될 때"
description: "Flutter 3.50 Web release에서 CSP 때문에 부트스트랩이나 WASM이 실행되지 않을 때, 외부 스크립트·nonce·wasm-unsafe-eval을 나누어 적용하는 운영 기준을 정리한다."
date: 2026-11-15
tags: [Flutter, Web, WASM, CI/CD, 출시]
comments: true
share: true
---

![Flutter Web release의 CSP와 WASM 로딩을 점검하는 브라우저 개발자 도구 화면](https://images.unsplash.com/photo-1551288049-bebda4e38f71?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 것은 release 시작 직전 브라우저 콘솔의 차단 리소스다.

Flutter Web을 CSP(Content-Security-Policy)와 함께 운영한다면 `flutter_bootstrap.js` 차단과 WASM 컴파일 차단을 따로 고쳐야 한다. 외부 부트스트랩만 쓰는 앱은 `script-src 'self'`와 `'wasm-unsafe-eval'`부터 확인하면 되고, `{{flutter_bootstrap_js}}`를 `index.html`에 인라인한 앱은 요청마다 바뀌는 nonce를 연결해야 한다. 보안 정책을 빠르게 통과시키려고 `'unsafe-eval'`과 `'unsafe-inline'`을 무조건 넣는 방식은 이 글의 대상이 아니다.

## 오류가 release에서만 보이는 이유

개발 서버에서는 실행되지만 배포 서버에서 아래 오류가 난다.

| 콘솔 메시지 | 원인 |
|---|---|
| `flutter_bootstrap.js` 차단 | `script-src`에 현재 origin 없음 |
| inline script 차단 | bootstrap에 nonce·hash 없음 |
| `WebAssembly.compileStreaming` 차단 | `'wasm-unsafe-eval'` 없음 |
| 엔진 style 차단 | loader에 nonce 미전달 |

Flutter 공식 초기화 기준으로 커스텀 bootstrap에는 `{{flutter_js}}`, `{{flutter_build_config}}`, `_flutter.loader.load()`가 필요하다.

## 외부 bootstrap과 인라인 bootstrap 구분

HTML마다 nonce를 넣기 어려운 정적 호스팅은 외부 script 방식을 쓴다.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; style-src 'self'; connect-src 'self' https: wss:; object-src 'none'
```

API·CDN 주소는 실제 사용하는 호스트만 남긴다. `'wasm-unsafe-eval'`은 `'unsafe-eval'`보다 범위가 좁다.

인라인 `{{flutter_bootstrap_js}}`가 필요하면 서버가 요청마다 nonce를 만들고 CSP, HTML, Flutter loader에 같은 값을 전달한다.

```html
<meta name="flutter-csp-nonce" content="{{NONCE}}">
<script nonce="{{NONCE}}">
  {{flutter_js}}
  {{flutter_build_config}}
  const nonce = document.querySelector('meta[name="flutter-csp-nonce"]').content;
  _flutter.loader.load({ config: { nonce } });
</script>
```

고정 nonce는 재사용하지 않는다. `{{flutter_build_config}}`를 빼거나 `main.dart.js`를 하드코딩하면 해시 빌드에서 entrypoint를 찾지 못한다.

## 배포 직전 체크리스트

- [ ] `flutter build web --wasm` 뒤 `flutter_bootstrap.js`가 200이다.
- [ ] HTML nonce, CSP nonce, `config.nonce`가 요청마다 일치한다.
- [ ] WASM 대상 브라우저에 `'wasm-unsafe-eval'`이 허용된다.
- [ ] `connect-src`에 API·WebSocket 주소가 있다.
- [ ] release 콘솔에서 최초 차단 리소스를 확인했다.

외부·인라인 bootstrap의 조건을 나누고, WASM에는 `'wasm-unsafe-eval'`, 인라인 요소에는 요청 단위 nonce를 적용하면 된다. nonce 주입이 어려우면 인라인을 제거한다.

공식 기준은 [Flutter Web 앱 초기화 문서](https://docs.flutter.dev/platform-integration/web/initialization), [Flutter WebAssembly 문서](https://docs.flutter.dev/platform-integration/web/wasm), [MDN CSP 헤더 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy)다.
