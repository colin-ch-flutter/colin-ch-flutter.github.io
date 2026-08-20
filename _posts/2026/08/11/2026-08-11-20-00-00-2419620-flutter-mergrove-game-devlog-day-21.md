---
layout: post
title: "스크린샷도 테스트로 찍는다고? 직접 해보니"
description: "Flutter integration_test로 Play 스토어용 게임 스크린샷을 일관되게 준비한 과정."
date: 2026-08-11
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1753715613831-9e48eac86813?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Jakub Żerdzicki](https://unsplash.com/es/fotos/una-persona-codifica-mientras-toma-notas-QUtrcUo5-GI)

수동 캡처는 점수와 타일 배치가 매번 달라져 스토어 원고를 맞추기 어렵다. 정해진 보드·점수·테마 상태를 주입하는 통합 테스트를 만들어 과수원 플레이, 컬렉션, 큰 보드를 반복 캡처했다.

스크린샷은 장식이 아니라 게임의 약속이다. 실제 앱 화면만 사용하고, 첫 세 장이 핵심 루프를 바로 보여주도록 순서를 정했다.


통합 테스트 캡처에는 화면 크기, 언어, 초기 보드와 점수를 고정했다. 같은 이미지가 다시 나와야 스토어 문구를 바꿔도 비교가 가능하고, 실제 UI 변경도 빨리 잡힌다.

스크린샷을 수동으로 찍을 때마다 점수와 타일이 달라졌다. 보기 좋은 한 장보다 같은 장면을 다시 만들 수 있는지가 더 중요하다는 걸 배웠다.

## 참고 자료

[Firebase for Flutter 설정](https://firebase.google.com/docs/flutter/setup) · [Firebase Remote Config 공식 안내](https://firebase.google.com/docs/remote-config/flutter/get-started)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
