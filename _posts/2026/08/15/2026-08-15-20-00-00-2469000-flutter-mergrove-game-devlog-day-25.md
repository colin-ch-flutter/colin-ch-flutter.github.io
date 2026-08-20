---
layout: post
title: "테스터 12명 모았다고 끝이 아니었다… 피드백 지옥"
description: "12인 비공개 테스트 피드백을 재현 가능성 기준으로 정리한 방법."
date: 2026-08-15
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1753715613831-9e48eac86813?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Jakub Żerdzicki](https://unsplash.com/es/fotos/una-persona-codifica-mientras-toma-notas-QUtrcUo5-GI)

메시지 한 줄을 바로 기능 요청으로 바꾸면 우선순위가 흔들린다. 피드백은 설치 문제, 규칙 이해, 화면 읽기, 저장·오류, 광고 경험으로 나누고 기기·보드 크기·재현 절차를 함께 받았다.

한 명의 취향과 반복되는 문제를 구분했다. 특히 첫 스와이프와 튜토리얼, 작은 화면의 버튼은 로그보다 사람의 관찰이 빨리 알려주는 영역이었다.


피드백 양식에는 ‘문제가 있었나요?’ 대신 기기 모델, Android 버전, 마지막 행동, 스크린샷을 요청했다. 한 줄 피드백도 재현 조건이 붙으면 수정 우선순위를 정할 수 있다.

피드백이 많이 오면 좋기만 할 줄 알았다. 비슷한 말이 반복되는지, 특정 기기에서만 생기는지 나눠 보니 그제야 고칠 순서가 보였다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
