---
layout: post
title: "Flutter Web Firebase Hosting 404 오류 - PathUrlStrategy 새로고침과 base href 복구"
description: "Flutter Web을 Firebase Hosting에 배포한 뒤 내부 경로 새로고침에서 404가 나는 원인을 PathUrlStrategy, firebase.json rewrite, base href 조건으로 나눠 복구하는 방법을 정리했다."
date: 2026-09-18
tags: [Flutter, Firebase, 웹, 배포, CI/CD]
comments: true
share: true
---

![Flutter Web 앱의 Firebase Hosting 경로 새로고침 404를 점검하는 배포 설정 화면](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&q=80)

Flutter Web 내부 경로를 주소창에 직접 열거나 새로고침할 때 Firebase Hosting 404가 난다면 `PathUrlStrategy`를 없애기보다 `firebase.json`의 rewrite와 `base href`를 배포 위치에 맞추는 선택이 유리하다. 루트 도메인에 정적 페이지만 올렸거나 해시 URL(`#`)을 유지해도 되는 앱에는 이 설정이 필요 없다.

## 로컬에서는 되는데 배포 후에만 404인 이유

Flutter Web의 `PathUrlStrategy`는 `/settings` 같은 경로를 History API로 만든다. 앱 안에서 버튼을 눌러 이동하면 브라우저가 이미 받은 `index.html`이 라우팅을 처리한다. 그러나 새로고침하면 서버에 `/settings` 파일을 요청한다. 해당 파일이 없는데 서버가 `index.html`로 rewrite하지 않으면 Firebase의 404가 먼저 반환된다.

| 증상 | 실제로 확인할 위치 | 우선 조치 |
|---|---|---|
| 앱 안의 이동만 성공 | `main.dart` | `usePathUrlStrategy()` 여부 확인 |
| `/settings` 새로고침 404 | `firebase.json` | 모든 미존재 경로를 `/index.html`로 rewrite |
| 서브경로에서 JS·아이콘 404 | `web/index.html` | `<base href>`를 서브경로로 변경 |
| 배포 직후에도 예전 화면 | Hosting 캐시·서비스워커 | 새 빌드와 실제 응답을 분리 점검 |

## 루트 도메인 배포 설정

Flutter SDK에 포함된 URL 전략을 앱 시작 전에 켜야 경로 URL을 사용한다. 해시 URL이 괜찮다면 이 코드를 추가하지 않는 것이 무료이고 단순한 대안이다.

```dart
import 'package:flutter_web_plugins/url_strategy.dart';

void main() {
  usePathUrlStrategy();
  runApp(const MyApp());
}
```

루트 도메인(`https://example.web.app/`)이라면 `web/index.html`의 기본 경로는 `/`로 둔다.

```html
<base href="/">
```

`build/web`을 Firebase Hosting의 공개 디렉터리로 지정하고, 실제 파일이 없는 경로만 Flutter의 진입점으로 보낸다. Firebase는 rewrite보다 실제 정적 파일을 우선하므로 `main.dart.js` 같은 자산 요청까지 무조건 가로채지는 않는다.

```json
{
  "hosting": {
    "public": "build/web",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {"source": "**", "destination": "/index.html"}
    ]
  }
}
```

배포 순서는 설정이 적용된 빌드를 만든 뒤 같은 디렉터리를 올리는 방식으로 고정한다.

```bash
flutter build web --release
firebase deploy --only hosting
```

## `/dashboard/` 같은 서브경로에 올릴 때

Firebase Hosting 사이트 전체가 아니라 reverse proxy나 별도 도메인의 `/dashboard/` 아래에 앱을 배치하면 `/`를 그대로 쓰면 안 된다. `web/index.html`을 아래처럼 바꾸고 빌드한다.

```html
<base href="/dashboard/">
```

이때 서버 rewrite는 앱 경로의 “페이지 요청”에만 적용해야 한다. `/dashboard/main.dart.js`까지 `/index.html`로 바꾸면 브라우저가 JavaScript 대신 HTML을 받아 부팅에 실패한다. Firebase Hosting 단독으로 하위 디렉터리 배포를 구성할 때는 실제 공개 디렉터리 구조와 destination을 함께 검증하고, reverse proxy를 쓴다면 아래처럼 자산과 페이지를 분리하는 규칙이 필요하다.

```text
/dashboard/assets/**       → build/web/assets/**
/dashboard/*.js, *.wasm    → build/web의 해당 파일
/dashboard/경로(확장자 없음) → build/web/index.html
```

`base href`만 바꾸고 이 매핑을 생략하면 내부 이동은 되지만 새로고침은 계속 실패한다. 반대로 루트 도메인에 `build/web`을 그대로 Firebase Hosting하는 경우에는 `base href="/"`와 앞의 `source: "**"` rewrite가 가장 단순하다.

## 배포 직후 확인할 체크리스트

- [ ] `flutter build web --release` 후 `build/web/index.html`의 base 경로를 확인했다.
- [ ] `firebase.json`의 `public`이 `build/web`을 가리킨다.
- [ ] 앱 내부 링크와 주소창 직접 입력을 각각 시험했다.
- [ ] `/settings`, `/settings/`처럼 trailing slash가 다른 URL도 확인했다.
- [ ] Chrome 개발자 도구에서 `main.dart.js`, `flutter_bootstrap.js`가 404가 아닌지 확인했다.
- [ ] `curl -I https://도메인/실제경로`에서 호스팅 응답이 404인지, 앱 내부의 unknown route인지 구분했다.

핵심은 세 값의 일치다. 앱이 생성하는 URL 전략은 `PathUrlStrategy`, HTML의 기준 경로는 `base href`, 서버의 실패 경로 처리는 `rewrites`가 담당한다. Flutter 공식 문서도 Path URL 전략에는 서버가 `index.html`로 rewrite해야 한다고 설명하고, Firebase Hosting 문서는 SPA용 rewrite 예시와 규칙 우선순위를 제공한다. 셋 중 하나만 빠져도 로컬 성공과 운영 404가 동시에 나타난다.

참고 문서: [Flutter URL 전략 공식 문서](https://docs.flutter.dev/ui/navigation/url-strategies), [Flutter Web 배포 문서](https://docs.flutter.dev/deployment/web), [Firebase Hosting rewrite 설정](https://firebase.google.com/docs/hosting/full-config)
