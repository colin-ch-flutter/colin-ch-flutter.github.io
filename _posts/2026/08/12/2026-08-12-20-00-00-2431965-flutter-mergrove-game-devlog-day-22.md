---
layout: post
title: "아이콘에 글자 넣지 마세요… 결국 다 지웠다"
description: "Play 스토어용 Flutter 게임 아이콘과 feature graphic을 검토한 기준."
date: 2026-08-12
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-22-tablet_10_store_gameplay_vegetable_4x4.png)

작은 아이콘에 게임 이름·점수·문구를 모두 넣으면 목록에서 읽히지 않는다. 과일 타일의 형태와 색 대비만 남기고, 텍스트와 과장된 배지는 제거했다.

1024×500 기능 그래픽도 중앙 안전 영역에만 중요한 요소를 두었다. 스토어 이미지는 화면의 모든 정보를 전달하기보다 설치 전에 분위기와 장르를 빠르게 알려야 한다.


아이콘은 512px 정사각형에서 먼저 확인했다. 큰 화면에서 예쁜 미세한 요소는 Play 스토어 목록 크기에서는 지저분한 점으로 보일 수 있어 실루엣과 대비만 남겼다.

아이콘에 설명을 더할수록 작은 크기에서는 아무것도 안 보였다. 아까워도 글자를 지우고 형태만 남겼을 때 훨씬 또렷해졌다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
