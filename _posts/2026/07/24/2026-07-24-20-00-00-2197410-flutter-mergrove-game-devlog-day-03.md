---
layout: post
title: "UI부터 만들면 망한다? 게임 엔진을 먼저 꺼낸 이유"
description: "Flutter 2048 게임의 이동과 병합 규칙을 UI에서 분리해 테스트 가능하게 만든 구조."
date: 2026-07-24
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-03-gameplay-hero.png)

스와이프 처리와 병합 규칙을 한 위젯에 넣으면 애니메이션을 고칠 때 규칙도 흔들린다. `lib/game/bloom_engine.dart`를 순수 Dart 영역으로 두고, Flutter는 입력·화면·오버레이만 담당하게 분리했다.

이 구조 덕분에 4×4 보드의 한 줄 병합, 빈칸 압축, 점수 증가를 화면 없이 재현할 수 있었다. 게임의 재미는 UI보다 규칙의 예측 가능성에서 먼저 나온다.


엔진의 입력과 출력은 숫자 배열과 이동 방향뿐이다. `BuildContext`, 애니메이션, 파일 저장을 참조하지 않게 해 같은 보드 결과를 테스트에서 그대로 비교할 수 있게 했다.

화면을 먼저 예쁘게 만드는 유혹이 컸다. 그래도 숫자 배열이 틀리면 아무리 예쁜 타일도 결국 이상하게 움직인다는 걸 초반에 확인했다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
