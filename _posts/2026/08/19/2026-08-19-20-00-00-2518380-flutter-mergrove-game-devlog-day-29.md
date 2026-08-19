---
layout: post
title: "Flutter 2048 게임 개발기 29 - 공개 전날, 기능 추가를 멈춘 이유"
description: "Flutter 게임 Play 스토어 공개 전날에 버그 수정과 등록 정보만 다듬은 이유."
date: 2026-08-19
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/04_night_garden_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

마지막 날에는 좋은 아이디어가 가장 위험하다. 새 테마나 보상 기능은 미루고, AAB 서명, 패키지명, 개인정보처리방침 URL, 데이터 보안, 스토어 설명과 스크린샷 순서를 다시 확인했다.

## 오늘의 구현 판단

Remote Config의 광고 기본값은 보수적으로 두고, 장애가 나면 광고를 끌 수 있는지 확인했다. 공개 직전의 목표는 더 많은 기능이 아니라 설치·실행·한 판 완료가 안정적인 상태였다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

공개 직전에는 변경 기록을 줄여야 원인을 추적할 수 있다. AAB와 스토어 등록정보를 고정한 뒤에는 피드백이 있어도 긴급 오류가 아닌 이상 다음 버전 후보로 분리했다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/01_core_merge_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: Remote
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
