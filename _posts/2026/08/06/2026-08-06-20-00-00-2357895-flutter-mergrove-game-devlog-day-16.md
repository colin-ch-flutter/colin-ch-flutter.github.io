---
layout: post
title: "점수만 보면 지루했다… 도감 화면을 넣은 진짜 이유"
description: "발견·미발견 타일을 보여주는 Flutter 게임 컬렉션 화면 구현 판단."
date: 2026-08-06
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-16-store_gameplay_vegetable_4x4_compatible.png)

게임판만 있으면 최고 점수 외에는 남는 것이 적다. 발견한 타일과 아직 비어 있는 슬롯을 함께 보여줘 다음 목표를 자연스럽게 만들었다.

미발견 항목은 이름까지 모두 공개하지 않고 실루엣과 단계만 남겼다. 보상은 화려한 팝업보다 ‘어디까지 왔는지’를 확인하는 화면에서 오래 작동했다.


도감의 미발견 칸은 무작정 잠금 아이콘으로 채우지 않았다. 다음 단계가 있다는 신호는 주되, 이름과 완성 이미지는 발견 순간까지 남겨 보상의 밀도를 유지했다.

점수만 올리는 플레이는 금방 숫자 노동처럼 느껴졌다. 아직 못 본 칸 하나가 보이자 다시 한 판을 하고 싶어지는 이유가 생겼다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
