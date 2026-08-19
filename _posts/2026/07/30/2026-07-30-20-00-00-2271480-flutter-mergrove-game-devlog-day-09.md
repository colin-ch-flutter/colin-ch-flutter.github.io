---
layout: post
title: "되돌리기 무한으로 넣을까? 딱 한 번만 준 이유"
description: "Flutter 2048 게임의 undo 기능을 한 수로 제한해 전략성과 단순함을 맞춘 기록."
date: 2026-07-30
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-09-phone_gems_en.png)

무제한 되돌리기는 퍼즐의 긴장을 지우고, 저장할 상태도 복잡하게 만든다. 이동 전 보드·점수·발견 상태를 한 번만 복사해두고 유효한 이동 뒤에만 교체했다.

되돌리기 버튼이 비활성화되는 상태도 명확히 보여줬다. 이 한 수가 있으면 실수의 좌절은 줄지만, 매 순간을 계산해야 하는 2048의 핵심은 남는다.


되돌리기 저장본에는 타일 배열만이 아니라 현재 점수와 발견 목록 변경 전 상태가 필요했다. 둘 중 하나만 되돌리면 점수와 컬렉션이 어긋나는 버그가 생긴다.

실수했을 때 바로 앱을 닫고 싶어지는 순간이 있었다. 한 번의 되돌리기는 그 감정을 낮춰 주지만, 퍼즐을 너무 느슨하게 만들지는 않았다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
