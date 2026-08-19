---
layout: post
title: "Flutter 2048 게임 개발기 5 - 4×4, 5×5, 6×6 보드를 한 엔진으로 처리하기"
description: "Flutter 게임에서 보드 크기를 고정하지 않고 공통 엔진으로 확장한 기록."
date: 2026-07-26
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/05_ocean_life_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

4×4만 맞춘 뒤 보드 크기를 늘리면 행과 열 경계에서 하드코딩이 계속 나온다. 엔진은 `size`를 기준으로 행을 추출하고 회전·반전 없이 네 방향 이동을 계산하도록 만들었다.

## 오늘의 구현 판단

화면에서는 타일 크기만 가용 폭에서 다시 계산했다. 작은 화면에서 6×6이 너무 답답해지는 지점도 확인했고, 숫자보다 타일 그림의 대비가 더 중요하다는 사실을 일찍 알았다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

보드 크기를 바꾸는 테스트에서는 새 타일이 보드 바깥에 생기지 않는지와 빈 칸 수가 맞는지를 함께 봤다. 화면 크기 문제와 엔진의 경계 조건을 한 테스트에서 섞지 않는 것이 핵심이다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/02_garden_world_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 화면에서는
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
