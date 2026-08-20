---
layout: post
title: "테스트가 계속 깨져서… 엔진과 화면을 갈라버렸다"
description: "BloomEngine 단위 테스트와 Flutter 위젯 테스트를 나눈 방식."
date: 2026-08-04
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1753715613831-9e48eac86813?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Jakub Żerdzicki](https://unsplash.com/es/fotos/una-persona-codifica-mientras-toma-notas-QUtrcUo5-GI)

게임 규칙은 빠르게 수백 번 검증해야 하고, 화면은 실제 흐름을 검증해야 한다. 그래서 병합·점수·새 타일 생성은 순수 Dart 테스트로, 홈·테마 선택·튜토리얼은 위젯 테스트로 나눴다.

버그가 나면 먼저 엔진인지 화면인지 범위를 줄일 수 있었다. 테스트가 느리면 개발 중에 실행하지 않게 되므로, 경계를 나눈 선택이 가장 실용적이었다.


엔진 단위 테스트는 빠른 실패 신호를 주고, 위젯 테스트는 실제 버튼과 화면 문구가 바뀌지 않았는지 알려준다. 스토어용 캡처는 통합 테스트로 분리해 재현성을 확보했다.

테스트가 실패했을 때 화면부터 의심하면 시간이 너무 많이 샜다. 규칙이 틀렸는지, 버튼이 틀렸는지 따로 말해 주는 테스트가 필요했다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
