---
layout: post
title: "Flutter 2048 게임 개발기 3 - 순수 Dart 엔진부터 분리한 이유"
description: "Flutter 2048 게임의 이동과 병합 규칙을 UI에서 분리해 테스트 가능하게 만든 구조."
date: 2026-07-24
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/03_collect_discover_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

스와이프 처리와 병합 규칙을 한 위젯에 넣으면 애니메이션을 고칠 때 규칙도 흔들린다. `lib/game/bloom_engine.dart`를 순수 Dart 영역으로 두고, Flutter는 입력·화면·오버레이만 담당하게 분리했다.

## 오늘의 구현 판단

이 구조 덕분에 4×4 보드의 한 줄 병합, 빈칸 압축, 점수 증가를 화면 없이 재현할 수 있었다. 게임의 재미는 UI보다 규칙의 예측 가능성에서 먼저 나온다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

엔진의 입력과 출력은 숫자 배열과 이동 방향뿐이다. `BuildContext`, 애니메이션, 파일 저장을 참조하지 않게 해 같은 보드 결과를 테스트에서 그대로 비교할 수 있게 했다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/05_ocean_life_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 이
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
