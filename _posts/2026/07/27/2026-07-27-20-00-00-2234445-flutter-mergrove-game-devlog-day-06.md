---
layout: post
title: "숫자만 합치는데 재미가 없었다… 과일 타일로 갈아엎은 이유"
description: "Flutter 2048 게임의 숫자 타일을 과일 컬렉션으로 바꾼 디자인 판단."
date: 2026-07-27
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-06-phone_5x5_en.png)

숫자만 보이는 2048은 규칙은 즉시 이해되지만 긴 플레이의 보상이 약했다. 2부터 65,536까지 과일 이름과 타일 이미지를 연결해, 다음 병합이 점수뿐 아니라 새 발견이 되게 만들었다.

단, 숫자를 완전히 지우지는 않았다. 접근성과 학습성을 위해 값은 화면에 남기고, 이미지와 색은 값을 보조하는 역할로 제한했다.


자산 이름도 값과 분리했다. 같은 128이라도 테마에 따라 사과·꽃·보석이 될 수 있으므로, 점수 계산은 숫자로 하고 표현만 메타데이터가 맡도록 했다.

숫자 타일만 놓고 한 판을 해 봤는데, 목표를 찍어도 감정이 거의 남지 않았다. 과일 하나가 바뀌는 것만으로도 다음 수를 보는 이유가 생겼다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
