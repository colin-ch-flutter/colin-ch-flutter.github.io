---
layout: post
title: "Flutter 2048 게임 개발기 21 - 플레이 스토어 스크린샷을 테스트로 만든 이유"
description: "Flutter integration_test로 Play 스토어용 게임 스크린샷을 일관되게 준비한 과정."
date: 2026-08-11
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/01_core_merge_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

수동 캡처는 점수와 타일 배치가 매번 달라져 스토어 원고를 맞추기 어렵다. 정해진 보드·점수·테마 상태를 주입하는 통합 테스트를 만들어 과수원 플레이, 컬렉션, 큰 보드를 반복 캡처했다.

## 오늘의 구현 판단

스크린샷은 장식이 아니라 게임의 약속이다. 실제 앱 화면만 사용하고, 첫 세 장이 핵심 루프를 바로 보여주도록 순서를 정했다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

통합 테스트 캡처에는 화면 크기, 언어, 초기 보드와 점수를 고정했다. 같은 이미지가 다시 나와야 스토어 문구를 바꿔도 비교가 가능하고, 실제 UI 변경도 빨리 잡힌다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/03_collect_discover_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Firebase for Flutter 설정](https://firebase.google.com/docs/flutter/setup) · [Firebase Remote Config 공식 안내](https://firebase.google.com/docs/remote-config/flutter/get-started)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 스크린샷은
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
