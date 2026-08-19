---
layout: post
title: "Flutter 2048 게임 개발기 19 - Firebase를 넣되 오프라인 플레이를 지킨 방법"
description: "Flutter 게임에서 Analytics·Crashlytics·Remote Config를 운영 도구로 쓰는 방식."
date: 2026-08-09
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/04_night_garden_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

계정 없이 오프라인으로 즐기는 게임이 목표였지만, 출시 뒤의 오류와 광고 정책은 관찰할 방법이 필요했다. Firebase Analytics와 Crashlytics는 안정성 확인에, Remote Config는 최소 지원 버전과 광고 정책에만 사용했다.

## 오늘의 구현 판단

핵심 보드 상태는 기기에 남기고, 원격 연결 실패가 게임 시작을 막지 않게 안전한 기본값을 두었다. 운영 도구가 게임의 의존성이 되지 않도록 범위를 좁혔다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 핵심
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
