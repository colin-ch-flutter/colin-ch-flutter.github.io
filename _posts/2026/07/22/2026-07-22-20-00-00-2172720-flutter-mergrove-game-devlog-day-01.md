---
layout: post
title: "30일이면 되겠지? 2048 게임 기획부터 다시 멈춰 세운 이유"
description: "Flutter로 2048 게임 Mergrove를 기획하며 30일 개발 범위와 출시 기준을 정리한 기록."
date: 2026-07-22
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://p0.piqsels.com/preview/60/688/850/business-office-graphic-designer-planning.jpg)

[이미지 출처: Piqsels 공개 이미지](https://www.piqsels.com/en/public-domain-photo-zkwuh)

클래식 2048을 그대로 복제하지 않고, 같은 타일을 합치면 과일·꽃·보석 발견으로 이어지는 차분한 퍼즐로 방향을 잡았다. 이번 첫날에는 ‘한 손 조작’, ‘계정 없이 시작’, ‘오프라인 기본 플레이’ 세 가지를 고정했다.

기획 문서에는 4×4 기본판, 5×5 전략판, 6×6 장기 플레이를 적었다. 기능을 늘리기보다 한 판을 다시 시작하고 싶어지는 감각을 먼저 검증하기로 했다.


기획 단계에서는 ‘완료’의 정의를 앱 실행이 아니라 스와이프 한 번으로 규칙을 이해하고, 종료 뒤 다시 시작할 이유가 남는 상태로 잡았다. 이 기준이 이후의 저장·컬렉션·스토어 이미지 판단을 줄 세우는 기준이 됐다.

처음에는 ‘게임 하나니까 빨리 끝나겠지’라고 생각했다. 그런데 기획을 적어 보니, 무엇을 안 만들지 정하는 일이 타일을 만드는 일보다 훨씬 오래 걸렸다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
