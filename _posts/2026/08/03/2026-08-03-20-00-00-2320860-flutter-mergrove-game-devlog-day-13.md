---
layout: post
title: "진동이 좋다고 다 좋은 건 아니었다"
description: "Flutter 게임의 햅틱 피드백과 reduced motion 옵션을 함께 설계한 기록."
date: 2026-08-03
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-13-store_collection_orchard_complete.png)

병합 순간의 작은 진동은 손맛을 살리지만 모든 사람에게 편한 기능은 아니다. 설정에서 햅틱과 모션을 따로 켜고 끌 수 있게 하고, 애니메이션을 줄여도 결과와 입력 순서는 변하지 않게 했다.

접근성은 출시 직전에 붙이는 장식이 아니었다. 작은 화면에서 글자가 읽히고, 움직임이 부담스러운 이용자도 한 판을 끝낼 수 있는지가 기본 품질 기준이 됐다.


모션 줄이기 설정은 단순히 애니메이션 시간을 0으로 바꾸지 않았다. 완료 콜백 순서를 유지해야 저장과 오버레이 타이밍이 일반 설정과 동일하게 작동한다.

진동을 켜고 플레이하면 손맛이 살아났지만, 오래 해 보니 피곤할 수도 있겠다고 느꼈다. 내가 좋아하는 효과를 모두에게 기본값으로 강요하면 안 됐다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
