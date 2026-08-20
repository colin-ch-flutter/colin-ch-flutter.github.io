---
layout: post
title: "광고 제거 버튼 하나가 이렇게 피곤할 줄이야"
description: "Flutter in_app_purchase로 비소모성 광고 제거 상품을 연결할 때의 기준."
date: 2026-08-08
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-18-tablet_10_orchard.png)

광고 제거는 반복 구매가 아닌 한 번의 권한이어야 한다. 상품 ID를 `remove_ads`로 고정하고, 구매 복원과 로컬 반영을 분리해 처리했다.

스토어 상품이 아직 준비되지 않은 환경에서는 구매 버튼이 실패하지 않고 안내 상태로 남아야 한다. 결제 기능은 성공 화면보다 실패·복원·오프라인 상태를 먼저 설계해야 한다.


광고 제거 구매는 기기 변경 뒤에도 복원할 수 있어야 한다. 구매 스트림을 처리할 때 보류·실패·복원 상태를 구분하고, UI는 완료 신호를 받은 뒤에만 광고를 끈다.

구매 버튼은 만들기 쉽고, 구매가 애매하게 끝났을 때 안내하는 일은 어렵다. 결제가 아니라 불안한 순간을 다루는 기능이라고 생각하게 됐다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
