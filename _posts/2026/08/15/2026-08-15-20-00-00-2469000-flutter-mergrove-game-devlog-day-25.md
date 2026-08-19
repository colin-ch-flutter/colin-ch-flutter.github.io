---
layout: post
title: "Flutter 2048 게임 개발기 25 - Tester Community에서 받은 피드백을 분류하는 법"
description: "12인 비공개 테스트 피드백을 재현 가능성 기준으로 정리한 방법."
date: 2026-08-15
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/05_ocean_life_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

메시지 한 줄을 바로 기능 요청으로 바꾸면 우선순위가 흔들린다. 피드백은 설치 문제, 규칙 이해, 화면 읽기, 저장·오류, 광고 경험으로 나누고 기기·보드 크기·재현 절차를 함께 받았다.

## 오늘의 구현 판단

한 명의 취향과 반복되는 문제를 구분했다. 특히 첫 스와이프와 튜토리얼, 작은 화면의 버튼은 로그보다 사람의 관찰이 빨리 알려주는 영역이었다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

피드백 양식에는 ‘문제가 있었나요?’ 대신 기기 모델, Android 버전, 마지막 행동, 스크린샷을 요청했다. 한 줄 피드백도 재현 조건이 붙으면 수정 우선순위를 정할 수 있다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/02_garden_world_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 한
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
