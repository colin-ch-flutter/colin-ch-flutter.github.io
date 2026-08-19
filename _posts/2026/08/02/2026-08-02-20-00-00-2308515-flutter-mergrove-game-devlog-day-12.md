---
layout: post
title: "2048 찍고 끝? 여기서 끝내기 싫었던 이유"
description: "Flutter 2048 게임에서 목표 달성 후 플레이를 이어가게 한 오버레이 설계."
date: 2026-08-02
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-12-phone_vegetable_en.png)

2,048 도달을 게임 종료로 처리하면 컬렉션의 높은 단계가 의미 없어졌다. 첫 bloom 도달에는 축하 오버레이를 띄우되, 계속하기와 새 게임 선택을 함께 제공했다.

4,096 이후도 같은 패턴으로 확장할 수 있게 상태를 분리했다. 화면 위에 오버레이가 있을 때는 광고나 새 입력이 들어오지 않도록 막은 점도 출시 전에는 꼭 확인했다.


마일스톤 오버레이는 결과를 확인하기 전까지 입력을 막는다. 축하 화면이 뜬 프레임에 새 이동이나 광고 요청이 겹치면 상태 복원과 분석 이벤트가 틀어질 수 있다.

2,048을 만들고 바로 끝내면 축하받은 느낌보다 뺏긴 느낌이 컸다. 그래서 ‘계속할래?’를 묻는 화면은 생각보다 중요한 결정이 됐다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
