---
layout: post
title: "내 폰에서는 괜찮았는데? 375px에서 다 무너진 화면"
description: "작은 휴대폰 화면에서 Flutter 2048 보드의 읽기성을 잡은 방법."
date: 2026-08-05
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://p0.piqsels.com/preview/60/688/850/business-office-graphic-designer-planning.jpg)

[이미지 출처: Piqsels 공개 이미지](https://www.piqsels.com/en/public-domain-photo-zkwuh)

에뮬레이터의 넓은 화면에서는 예뻤지만 375×812 크기에서는 상단 점수와 하단 버튼이 보드를 밀어냈다. 고정 타일 크기를 버리고 화면 폭·여백·보드 크기에서 계산하도록 바꿨다.

4×4와 6×6의 폰트 크기도 같게 두지 않았다. 컴팩트 조건에서는 정보량보다 터치와 식별을 우선했다.


작은 화면 검증에서는 최소 여백보다 타일 간 간격을 먼저 고정했다. 타일이 서로 붙으면 색이 좋아도 병합 가능한 대상이 한눈에 안 들어오기 때문이다.

큰 화면에서만 보던 레이아웃은 너무 자신만만했다. 실제 작은 화면을 보자마자 ‘이건 내가 편하게 보려고 만든 화면이구나’ 싶었다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
