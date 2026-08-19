---
layout: post
title: "Flutter 2048 게임 개발기 13 - 햅틱과 모션 줄이기를 같이 넣은 이유"
description: "Flutter 게임의 햅틱 피드백과 reduced motion 옵션을 함께 설계한 기록."
date: 2026-08-03
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/03_collect_discover_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

병합 순간의 작은 진동은 손맛을 살리지만 모든 사람에게 편한 기능은 아니다. 설정에서 햅틱과 모션을 따로 켜고 끌 수 있게 하고, 애니메이션을 줄여도 결과와 입력 순서는 변하지 않게 했다.

## 오늘의 구현 판단

접근성은 출시 직전에 붙이는 장식이 아니었다. 작은 화면에서 글자가 읽히고, 움직임이 부담스러운 이용자도 한 판을 끝낼 수 있는지가 기본 품질 기준이 됐다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

모션 줄이기 설정은 단순히 애니메이션 시간을 0으로 바꾸지 않았다. 완료 콜백 순서를 유지해야 저장과 오버레이 타이밍이 일반 설정과 동일하게 작동한다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/05_ocean_life_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 접근성은
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
