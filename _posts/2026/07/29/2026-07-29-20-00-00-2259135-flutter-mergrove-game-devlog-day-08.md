---
layout: post
title: "Flutter 2048 게임 개발기 8 - 스와이프 입력을 보드 좌표가 아닌 방향으로 다루기"
description: "GestureDetector로 2048 스와이프 방향을 안정적으로 판정한 과정."
date: 2026-07-29
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/03_collect_discover_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

짧은 탭과 대각선 드래그가 섞이면 의도하지 않은 이동이 자주 생긴다. 시작점과 종료점의 가로·세로 차이를 비교하고, 일정 거리보다 작은 제스처는 무시하도록 했다.

## 오늘의 구현 판단

입력은 `up`, `down`, `left`, `right` 네 방향으로만 엔진에 전달했다. UI가 보드 내부 배열을 직접 만지지 않으니 제스처 임계값을 바꿔도 병합 테스트는 그대로 유지됐다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

스와이프 임계값은 실제 화면 밀도와 관계없이 체감이 일정해야 했다. 대각선 이동은 더 큰 축만 채택하고, 임계값 아래의 움직임은 무효 처리해 스크롤과 오조작을 줄였다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/05_ocean_life_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 입력은
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
