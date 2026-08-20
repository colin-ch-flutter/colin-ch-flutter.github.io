---
layout: post
title: "광고 한 번 잘못 띄우면 게임이 끝난다"
description: "Flutter 게임 전면 광고를 플레이 흐름을 해치지 않게 제한한 방법."
date: 2026-08-07
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-17-store_home.png)

광고 수익을 먼저 생각하면 한 판의 리듬이 망가지기 쉽다. Mergrove에서는 게임 종료 뒤에만 전면 광고를 후보로 두고, 튜토리얼·병합 애니메이션·마일스톤 오버레이 중에는 절대 표시하지 않게 했다.

첫 두 번의 완료 게임은 광고 없이 두고, 이후에도 일정 횟수 이상 연속 노출하지 않는 정책을 세웠다. 원격 설정으로 즉시 끌 수 있는 탈출구도 필요했다.


광고 요청 자체도 게임 종료 뒤에만 준비하고, 실제 표시 직전 상태를 다시 검사했다. 광고 SDK가 준비됐다는 사실과 이용자에게 보여도 되는 순간은 다른 조건이다.

광고가 뜨는 위치를 바꿔 보며 직접 한 판을 끝냈다. 보상도 아닌데 흐름을 끊는 순간이 너무 선명해서, 수익보다 먼저 지켜야 할 선이 보였다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
