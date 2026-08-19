---
layout: post
title: "2주면 끝날 줄 알았는데… 일정표를 다시 찢은 날"
description: "Flutter 게임 개발에서 30일 계획을 기능·테스트·스토어 준비까지 나눈 방법."
date: 2026-07-23
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-02-merge-burst.png)

게임 규칙만 만들면 끝날 것 같았지만, 실제 출시에는 저장·접근성·개인정보·스크린샷·테스트 모집이 함께 필요했다. 개발 18일, 품질과 스토어 8일, 비공개 테스트와 피드백 4일로 쪼갰다.

날짜만 채우는 계획은 위험했다. 각 날에 ‘플레이 가능한 결과물’을 하나씩 남기고, 마지막 주에는 새 기능을 넣지 않는 규칙을 세웠다.


일정표에는 기능 목록과 별도로 위험 목록을 적었다. 플랫폼 설정, 개인 정보 문구, 테스터 참여 기간처럼 코드가 완성돼도 하루 안에 해결되지 않는 일은 앞쪽으로 당겼다.

달력에 작업을 넣어 놓으니 마음은 조금 편해졌다. 대신 하루가 밀렸을 때 어디를 덜어낼지까지 미리 적어 둔 것이 실제로 큰 도움이 됐다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
