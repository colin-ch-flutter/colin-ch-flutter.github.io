---
layout: post
title: "튜토리얼 3장 만들었다가 바로 지운 이유"
description: "Flutter 게임의 첫 실행 튜토리얼을 최소화해 이탈을 줄인 과정."
date: 2026-08-01
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1637563680361-3e7ee7599318?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Penfer](https://unsplash.com/photos/a-man-sitting-in-front-of-a-laptop-computer-_kbLtjx-pT0)

규칙 설명을 여러 장 넘기게 하면 퍼즐을 시작하기도 전에 닫아 버린다. 첫 게임에서 스와이프 방향과 같은 타일 병합만 보여주고, 실제 조작으로 바로 닫히게 만들었다.

튜토리얼 완료 여부는 로컬에 저장했다. 이미 아는 이용자에게 매번 설명을 강요하지 않는 것이 기능을 더 많이 보여주는 것보다 중요했다.


첫 화면에서 설명 문구를 길게 쓰지 않고 손가락이 움직여야 할 방향을 시각적으로 보여줬다. 실제 조작이 튜토리얼을 닫게 해 설명과 행동이 분리되지 않게 했다.

설명 화면을 길게 쓰면 친절해 보일 거라 착각했다. 내가 직접 세 장을 넘기고 나니 그냥 한 번 움직여 보는 편이 훨씬 낫겠다는 생각이 들었다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
