---
layout: post
title: "대각선으로 쓱 했는데 왜 움직여? 스와이프 오작동 잡기"
description: "GestureDetector로 2048 스와이프 방향을 안정적으로 판정한 과정."
date: 2026-07-29
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1637563680361-3e7ee7599318?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Penfer](https://unsplash.com/photos/a-man-sitting-in-front-of-a-laptop-computer-_kbLtjx-pT0)

짧은 탭과 대각선 드래그가 섞이면 의도하지 않은 이동이 자주 생긴다. 시작점과 종료점의 가로·세로 차이를 비교하고, 일정 거리보다 작은 제스처는 무시하도록 했다.

입력은 `up`, `down`, `left`, `right` 네 방향으로만 엔진에 전달했다. UI가 보드 내부 배열을 직접 만지지 않으니 제스처 임계값을 바꿔도 병합 테스트는 그대로 유지됐다.


스와이프 임계값은 실제 화면 밀도와 관계없이 체감이 일정해야 했다. 대각선 이동은 더 큰 축만 채택하고, 임계값 아래의 움직임은 무효 처리해 스크롤과 오조작을 줄였다.

내가 손가락을 비스듬히 긋는 습관이 있다는 것도 이때 알았다. 의도하지 않은 이동이 한 번 나오면 게임을 믿기 어려워져서 작은 입력도 그냥 넘길 수 없었다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
