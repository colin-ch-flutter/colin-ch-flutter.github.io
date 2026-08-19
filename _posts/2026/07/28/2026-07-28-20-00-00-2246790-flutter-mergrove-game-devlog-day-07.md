---
layout: post
title: "Flutter 2048 게임 개발기 7 - 테마 5개를 처음부터 데이터로 설계한 이유"
description: "Flutter 게임의 과수원·채소·꽃·숲·보석 테마를 확장 가능한 데이터로 정리한 방법."
date: 2026-07-28
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/02_garden_world_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

테마별 화면을 복사하면 이름 하나를 바꿀 때도 다섯 곳을 수정하게 된다. `GameTheme`과 타일 메타데이터를 기준으로 배경·색·이미지·컬렉션 이름을 선택하게 만들었다.

## 오늘의 구현 판단

처음부터 다섯 테마를 완성하려고 한 것은 아니다. 과수원 화면으로 루프를 먼저 확인한 뒤, 같은 규칙을 다른 분위기로 바꾸는 방식이 유지보수 비용을 낮췄다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

다섯 테마의 공통 규칙은 같은데 배경과 타일만 달라야 했다. 테마를 전환해도 저장 형식과 엔진이 바뀌지 않는지 확인하는 것이 새 테마를 추가할 때의 안전망이 됐다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/04_night_garden_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 처음부터
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
