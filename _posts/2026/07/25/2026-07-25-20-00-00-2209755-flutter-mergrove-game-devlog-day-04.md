---
layout: post
title: "2+2+2+2가 왜 안 되지? 첫 병합 버그에 멘붕한 날"
description: "Flutter 2048 병합 로직에서 같은 타일이 한 번만 합쳐지게 만든 방법."
date: 2026-07-25
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://p0.piqsels.com/preview/60/688/850/business-office-graphic-designer-planning.jpg)

[이미지 출처: Piqsels 공개 이미지](https://www.piqsels.com/en/public-domain-photo-zkwuh)

처음에는 인접한 숫자만 합치면 된다고 생각했다. 하지만 `[2, 2, 2, 2]`를 한 번 움직였을 때 결과가 `[4, 4]`여야 한다는 점에서 단순 반복문이 무너졌다.

한 줄을 압축한 뒤 왼쪽부터 한 번씩만 병합하고, 병합한 두 칸은 건너뛰는 방식으로 정리했다. 이동 결과·새 점수·움직인 타일 정보를 `MoveResult`로 함께 돌려 UI가 추측하지 않게 했다.


한 줄을 먼저 빈칸 없이 압축하고, 인접한 두 값을 한 번만 합친 뒤, 남은 칸을 다시 채운다. 이 순서를 명시하면 `[2,2,2]`가 `[4,2]`가 되는 예외도 설명 가능해진다.

테스트 배열을 손으로 적어 넣고 결과를 보는 순간이 꽤 처참했다. ‘당연히 되겠지’라고 넘긴 한 줄이 가장 오래 잡아먹었다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
