---
layout: post
title: "4×4만 했는데 왜 6×6까지? 보드 욕심의 결말"
description: "Flutter 게임에서 보드 크기를 고정하지 않고 공통 엔진으로 확장한 기록."
date: 2026-07-26
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-05-play-games-feature.png)

4×4만 맞춘 뒤 보드 크기를 늘리면 행과 열 경계에서 하드코딩이 계속 나온다. 엔진은 `size`를 기준으로 행을 추출하고 회전·반전 없이 네 방향 이동을 계산하도록 만들었다.

화면에서는 타일 크기만 가용 폭에서 다시 계산했다. 작은 화면에서 6×6이 너무 답답해지는 지점도 확인했고, 숫자보다 타일 그림의 대비가 더 중요하다는 사실을 일찍 알았다.


보드 크기를 바꾸는 테스트에서는 새 타일이 보드 바깥에 생기지 않는지와 빈 칸 수가 맞는지를 함께 봤다. 화면 크기 문제와 엔진의 경계 조건을 한 테스트에서 섞지 않는 것이 핵심이다.

보드 크기를 늘리자 갑자기 여백과 글자 크기가 다 싸우기 시작했다. 기능을 하나 추가한 것이 아니라 화면을 세 번 다시 만든 기분이었다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
