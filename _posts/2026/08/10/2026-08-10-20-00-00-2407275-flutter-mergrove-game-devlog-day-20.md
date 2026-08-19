---
layout: post
title: "Flutter 2048 게임 개발기 20 - 개인정보처리방침을 코드와 함께 검토한 이유"
description: "Flutter 게임의 Firebase·광고 SDK와 개인정보처리방침을 맞춘 출시 준비."
date: 2026-08-10
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/05_ocean_life_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

개인정보처리방침을 템플릿으로 끝내면 실제 SDK와 어긋나기 쉽다. 로컬 저장 데이터, Analytics·Crashlytics, Remote Config, 광고 동의 흐름을 코드 기준으로 대조했다.

## 오늘의 구현 판단

‘데이터를 수집하지 않는다’고 단정하지 않고, 광고와 진단 SDK가 처리할 수 있는 항목과 선택권을 명시했다. 스토어 선언은 최종 빌드와 실제 콘솔 설정으로 다시 확인해야 한다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: ‘데이터를
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
