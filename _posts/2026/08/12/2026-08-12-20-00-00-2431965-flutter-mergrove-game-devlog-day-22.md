---
layout: post
title: "Flutter 2048 게임 개발기 22 - 앱 아이콘과 기능 그래픽에서 버린 것들"
description: "Play 스토어용 Flutter 게임 아이콘과 feature graphic을 검토한 기준."
date: 2026-08-12
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-22-tablet_10_store_gameplay_vegetable_4x4.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

작은 아이콘에 게임 이름·점수·문구를 모두 넣으면 목록에서 읽히지 않는다. 과일 타일의 형태와 색 대비만 남기고, 텍스트와 과장된 배지는 제거했다.

## 오늘의 구현 판단

1024×500 기능 그래픽도 중앙 안전 영역에만 중요한 요소를 두었다. 스토어 이미지는 화면의 모든 정보를 전달하기보다 설치 전에 분위기와 장르를 빠르게 알려야 한다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

아이콘은 512px 정사각형에서 먼저 확인했다. 큰 화면에서 예쁜 미세한 요소는 Play 스토어 목록 크기에서는 지저분한 점으로 보일 수 있어 실루엣과 대비만 남겼다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 1024×500
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
