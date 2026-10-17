---
layout: post
title: "Flutter Web 배포 후 이전 화면이 보일 때 - PWA와 CDN 캐시 출시 조건"
description: "Flutter Web을 배포했는데 사용자가 이전 화면을 계속 볼 때 PWA 서비스 워커와 HTTP CDN 캐시를 분리해 확인하고, Firebase Hosting 헤더와 출시 체크리스트로 재발을 줄이는 방법을 정리한다."
date: 2026-10-17
tags: [Flutter, 배포, 성능최적화, CI/CD]
comments: true
share: true
---

![Flutter Web 배포 후 PWA 캐시 문제](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

Flutter Web을 배포했는데 일부 사용자에게 이전 화면이 계속 보인다면, 캐시를 무작정 지우기보다 PWA 서비스 워커와 브라우저·CDN의 HTTP 캐시를 나눠 확인해야 한다. 항상 최신 화면을 우선하는 사내 도구라면 `--pwa-strategy=none`과 짧은 `index.html` 캐시가 유리하고, 오프라인 지원이 필요한 앱은 서비스 워커를 유지하되 업데이트 지연을 출시 조건으로 관리해야 한다. 모바일 앱의 로컬 캐시 문제에는 이 글의 설정이 적용되지 않는다.

## 같은 “캐시”처럼 보여도 원인이 다르다

Flutter Web 배포 직후 화면이 바뀌지 않는 경로는 보통 세 가지다.

| 위치 | 남는 것 | 흔한 증상 | 확인할 값 |
|---|---|---|---|
| 서비스 워커 | PWA가 관리하는 정적 파일 캐시 | 새 배포 뒤에도 예전 JS 실행 | DevTools의 Application > Service Workers |
| 브라우저 HTTP 캐시 | `index.html`, 부트스트랩 파일 | 새로고침해도 같은 버전 | Network 응답의 `Age`, `Cache-Control` |
| CDN 엣지 | 서버 앞단의 응답 | 지역·사용자마다 화면이 다름 | 응답 헤더와 CDN purge 기록 |

Flutter 공식 Web FAQ 기준으로 Firebase Hosting의 정적 파일 기본 `Cache-Control`은 `max-age=3600`이다. 그러면 배포 후 최대 한 시간 동안 브라우저가 이전 `index.html`, `flutter_bootstrap.js`, manifest를 사용할 수 있다. 서비스 워커를 끄지 않았다면 HTTP 헤더를 바꿔도 이미 설치된 워커가 파일을 계속 제공할 수 있다.

그림에서 볼 부분은 `index.html`을 새로 받는 입구와, 그 뒤의 앱 파일을 오래 보관하는 경로를 분리하는 것이다.

```text
요청
  └─ index.html 재검증 ── 새 빌드 ID의 bootstrap 로드
                              └─ main.dart.js·assets는 해시 파일 캐시
```

## 최신 화면이 우선이면 PWA를 끈다

오프라인 실행이 제품 요구사항이 아니라면 생성 서비스 워커를 유지할 이유가 작다. 이 경우 배포 명령에서 PWA 전략을 끄고, 정적 파일을 새로 받도록 서버 헤더를 설정한다.

```bash
# 오프라인 PWA가 필요 없는 운영 도구의 예
flutter build web --release --pwa-strategy=none
```

Firebase Hosting을 쓴다면 `build/web` 안의 모든 파일을 장시간 캐시하기보다 진입 파일을 재검증하게 만든다. 아래 설정은 최신 배포 확인을 우선하는 예시다.

```json
{
  "hosting": {
    "public": "build/web",
    "headers": [
      {
        "source": "**/*.html",
        "headers": [
          { "key": "Cache-Control", "value": "max-age=0, must-revalidate" }
        ]
      },
      {
        "source": "flutter_bootstrap.js",
        "headers": [
          { "key": "Cache-Control", "value": "max-age=0, must-revalidate" }
        ]
      }
    ]
  }
}
```

이 설정은 배포 전 `firebase deploy`가 실제로 새 파일을 올렸다는 전제가 있다. `index.html`만 바뀌고 `main.dart.js`가 누락되면 캐시 문제가 아니라 불완전한 산출물 문제다. CI에서 `build/web`의 핵심 파일 존재 여부와 응답 상태 코드 200을 확인하는 편이 안전하다.

## 오프라인 PWA를 유지해야 하는 경우

서비스 워커가 필요한 앱은 “배포 즉시 모든 탭이 새 버전”이라고 가정하면 안 된다. 기존 탭이 열려 있으면 새 워커가 대기할 수 있고, 캐시된 앱 셸과 새 API가 잠시 섞일 수도 있다. API 응답을 깨뜨리는 변경은 웹 파일보다 먼저 호환 기간을 둬야 한다.

빌드 ID를 진단 정보로 남기면 “누가 어느 버전을 보고 있는가”를 확인하기 쉽다. Flutter FAQ처럼 정적 리소스 URL에 쿼리 버전이나 파일명 버전을 붙일 수 있지만, Flutter가 모든 리소스에 ID를 자동으로 붙이지는 않는다. CI 커밋 ID와 배포 로그를 함께 보관하는 편이 안전하다.

## 배포 직전 체크리스트

- [ ] 이번 앱이 오프라인 PWA를 실제로 요구하는지 결정했다.
- [ ] `--pwa-strategy=none` 사용 여부가 제품 요구사항과 일치한다.
- [ ] `index.html`과 `flutter_bootstrap.js`의 `Cache-Control`을 확인했다.
- [ ] CDN purge 또는 무효화가 필요한 호스팅인지 확인했다.
- [ ] 새 브라우저, 기존 서비스 워커가 있는 브라우저에서 각각 버전을 확인했다.
- [ ] `build/web`과 배포 서버의 파일 목록이 일치한다.
- [ ] 새 프런트엔드와 이전 API가 잠시 함께 실행돼도 깨지지 않는다.

캐시를 지우라는 안내는 임시 복구다. 최신 화면이 핵심이면 PWA를 제거하고 진입 파일을 재검증하게 한다. 오프라인이 핵심이면 서비스 워커를 남기되 버전 표시, 업데이트 안내, API 하위 호환을 함께 운영한다.

참고 문서: [Flutter Web FAQ](https://docs.flutter.dev/platform-integration/web/faq), [Flutter Web 배포](https://docs.flutter.dev/deployment/web)
