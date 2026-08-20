---
layout: post
title: "30일 만에 게임을 냈다… 그런데 진짜 시작은 지금"
description: "Flutter로 만든 2048 머지 퍼즐 Mergrove의 기획부터 Play 스토어 등록까지 30일 회고."
date: 2026-08-20
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-30-collection-source.png)

30일 동안 가장 크게 배운 것은 게임 엔진, 화면, 운영, 스토어 페이지, 테스터 모집이 분리된 일이 아니라 하나의 사용자 경험이라는 점이다. 순수 Dart 엔진으로 규칙을 지키고, Flutter로 입력과 화면을 만들고, 실제 테스트로 출시 기준을 확인했다.

Mergrove는 과일·꽃·보석을 합치는 4×4·5×5·6×6 퍼즐이다. Google Play에서 설치할 수 있으며, 홈페이지에는 실제 게임 화면과 테마 소개를 모아두었다.


출시 후에도 엔진 테스트와 실제 플레이 테스트의 역할은 끝나지 않는다. 충돌 보고·설치 흐름·비공개 테스트 피드백을 같은 분류로 남기면 다음 업데이트에서 감으로 고치지 않게 된다.

30일이 지나고 보니 출시 버튼을 누르는 순간보다, 그 뒤에 어떤 문제를 빨리 알아차릴지가 더 걱정됐다. 그래서 이 기록도 다음 업데이트의 시작점으로 남긴다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
