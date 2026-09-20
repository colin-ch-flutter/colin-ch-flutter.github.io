---
layout: post
title: "Flutter Web 배포 후 구버전 캐시 오류 - Firebase Hosting Cache-Control 점검"
description: "Flutter Web을 배포했는데 사용자가 구버전 화면을 볼 때, 서비스 워커와 브라우저·CDN 캐시를 구분하고 Firebase Hosting 설정과 검증 명령으로 재발을 막는 방법을 정리한다."
date: 2026-09-21
tags: [Flutter, Web, CI/CD, 성능최적화]
comments: true
share: true
---

![Flutter Web 배포 후 브라우저 캐시 점검](https://images.unsplash.com/photo-1555066931-4365d14bab8c?auto=format&fit=crop&w=1600&q=80)

이 그림에서는 새 배포 파일이 올라갔는데도 브라우저가 예전 JavaScript를 계속 읽는 상황을 봐야 한다.

Flutter Web을 배포했는데 일부 사용자만 구버전 화면을 본다면, `flutter clean`보다 `index.html`과 HTTP `Cache-Control`을 확인하는 선택이 유리하다. 현재 Flutter는 서비스 워커를 기본으로 생성·관리하지 않으므로, 별도 PWA 설정이 없다면 원인은 브라우저나 CDN 캐시일 가능성이 크다. 오프라인 기능이 필요하지 않은 앱은 서비스 워커를 새로 붙이지 않아도 해결할 수 있다.

## 증상이 사용자마다 다른 이유

Flutter Web은 `flutter_bootstrap.js`, `main.dart.js`, `canvaskit.wasm`, JSON 설정 파일을 브라우저에서 내려받는다. 이 중 하나라도 이전 배포본이면 화면은 열리지만 API 계약이나 라우트가 새 서버와 맞지 않을 수 있다. 특히 `Cache-Control: max-age=3600`이면 사용자는 배포 뒤 최대 1시간 동안 캐시된 파일을 볼 수 있다.

| 파일 종류 | 브라우저 캐시 | CDN 캐시 | 판단 기준 |
|---|---:|---:|---|
| `index.html` | 짧게 또는 금지 | 짧게 | 새 부트스트랩 주소를 발견해야 함 |
| `*.js`, `*.wasm`, `*.json` | `0` 권장 | 길게 허용 | HTML이 새 파일을 가리키면 재사용 가능 |
| 이미지·폰트·CSS | 1시간 이상 | 7일 정도 | 파일명이 바뀌지 않아도 영향이 작음 |

## Firebase Hosting 설정

앱 스크립트는 브라우저에서 다시 확인하게 하고, Firebase의 공유 CDN만 재사용하도록 `firebase.json`에 분리해서 적는다. 아래 값은 Flutter 공식 FAQ가 제시한 기준을 바탕으로 `index.html` 규칙을 추가한 예다.

```json
{
  "hosting": {
    "public": "build/web",
    "headers": [
      {
        "source": "**/index.html",
        "headers": [{
          "key": "Cache-Control",
          "value": "no-cache, no-store, must-revalidate"
        }]
      },
      {
        "source": "**/*.@(mjs|js|wasm|json)",
        "headers": [{
          "key": "Cache-Control",
          "value": "max-age=0,s-maxage=604800"
        }]
      },
      {
        "source": "**/*.@(png|jpg|jpeg|svg|webp|woff|woff2|css)",
        "headers": [{
          "key": "Cache-Control",
          "value": "max-age=3600,s-maxage=604800"
        }]
      }
    ]
  }
}
```

코드 변경 없이 서버 설정만 바꿨다면 배포 후 실제 응답 헤더를 확인해야 한다.

```bash
curl -I https://example.com/
curl -I https://example.com/flutter_bootstrap.js
curl -I https://example.com/main.dart.js
```

`index.html`에는 `no-store`가 보이고, JavaScript에는 브라우저 `max-age=0`과 CDN용 `s-maxage`가 각각 보여야 한다. 경로 앞에 CDN 도메인을 따로 사용한다면 앱이 실제로 읽는 도메인에도 같은 규칙을 적용해야 한다.

## 서비스 워커를 직접 쓰는 경우

Workbox나 예전 `flutter_service_worker.js`를 프로젝트에 남겨뒀다면 HTTP 헤더만 바꿔서는 부족하다. DevTools의 Application → Service Workers에서 등록 여부를 확인하고, 캐시 이름에 빌드 ID를 넣거나 새 워커가 `skipWaiting`·`clientsClaim` 정책을 갖는지 확인한다. 오프라인 지원이 없는 앱에서 서비스 워커가 남아 있다면 제거 후 일반 새로고침으로 재현 여부를 보는 편이 빠르다.

또 `web/index.html`이 오래된 템플릿이면 `serviceWorkerVersion`이나 `FlutterLoader.loadEntrypoint` deprecation 경고가 남을 수 있다. 최신 Flutter 기준 기본 진입은 `flutter_bootstrap.js`와 `_flutter.loader.load()`다. 경고를 숨기는 것보다 생성된 `build/web`에 실제로 어떤 파일이 들어갔는지 확인해야 한다.

## 배포 승인 체크리스트

- [ ] `flutter build web --release` 후 `build/web/index.html`이 새 빌드 파일을 가리킨다.
- [ ] HTML 응답의 브라우저 캐시가 배포 정책과 맞다.
- [ ] JS·Wasm·JSON 응답에 `max-age=0`이 적용된다.
- [ ] DevTools에서 오래된 서비스 워커와 Cache Storage가 없는지 확인했다.
- [ ] 새 배포 뒤 일반 새로고침과 시크릿 창에서 버전을 각각 확인했다.
- [ ] API 변경 시 구버전 앱도 임시로 호환되도록 서버를 배포했다.

짧게 정리하면, Flutter Web의 구버전 화면은 무조건 서비스 워커 문제로 보면 안 된다. `index.html`은 최신 진입점을 빨리 확인하게 하고, JS·Wasm·JSON은 브라우저 캐시를 짧게 둔 뒤 CDN만 재사용하는 구성이 운영에 맞다. 오프라인 PWA를 직접 운영하는 경우에만 서비스 워커 캐시 버전과 교체 정책을 별도로 관리한다.

참고 문서: [Flutter Web FAQ의 캐시·서비스 워커 안내](https://docs.flutter.dev/platform-integration/web/faq), [Flutter Web 초기화와 flutter_bootstrap.js](https://docs.flutter.dev/platform-integration/web/initialization), [Flutter Web 빌드·배포](https://docs.flutter.dev/deployment/web)
