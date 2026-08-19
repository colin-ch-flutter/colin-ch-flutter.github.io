---
layout: post
title: "Flutter 2048 게임 개발기 16 - 컬렉션 화면이 재방문 이유가 되게 만들기"
description: "발견·미발견 타일을 보여주는 Flutter 게임 컬렉션 화면 구현 판단."
date: 2026-08-06
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-16-store_gameplay_vegetable_4x4_compatible.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

게임판만 있으면 최고 점수 외에는 남는 것이 적다. 발견한 타일과 아직 비어 있는 슬롯을 함께 보여줘 다음 목표를 자연스럽게 만들었다.

## 오늘의 구현 판단

미발견 항목은 이름까지 모두 공개하지 않고 실루엣과 단계만 남겼다. 보상은 화려한 팝업보다 ‘어디까지 왔는지’를 확인하는 화면에서 오래 작동했다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

도감의 미발견 칸은 무작정 잠금 아이콘으로 채우지 않았다. 다음 단계가 있다는 신호는 주되, 이름과 완성 이미지는 발견 순간까지 남겨 보상의 밀도를 유지했다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 미발견
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
