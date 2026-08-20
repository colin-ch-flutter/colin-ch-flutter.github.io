---
layout: post
title: "테마 다섯 개? 처음엔 미친 짓인 줄 알았다"
description: "Flutter 게임의 과수원·채소·꽃·숲·보석 테마를 확장 가능한 데이터로 정리한 방법."
date: 2026-07-28
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-07-phone_6x6_en.png)

테마별 화면을 복사하면 이름 하나를 바꿀 때도 다섯 곳을 수정하게 된다. `GameTheme`과 타일 메타데이터를 기준으로 배경·색·이미지·컬렉션 이름을 선택하게 만들었다.

처음부터 다섯 테마를 완성하려고 한 것은 아니다. 과수원 화면으로 루프를 먼저 확인한 뒤, 같은 규칙을 다른 분위기로 바꾸는 방식이 유지보수 비용을 낮췄다.


다섯 테마의 공통 규칙은 같은데 배경과 타일만 달라야 했다. 테마를 전환해도 저장 형식과 엔진이 바뀌지 않는지 확인하는 것이 새 테마를 추가할 때의 안전망이 됐다.

처음에는 테마가 많을수록 풍성해 보일 거라고만 생각했다. 막상 관리할 데이터가 늘어나니, 공통 규칙을 끝까지 지키는 쪽이 더 중요했다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
