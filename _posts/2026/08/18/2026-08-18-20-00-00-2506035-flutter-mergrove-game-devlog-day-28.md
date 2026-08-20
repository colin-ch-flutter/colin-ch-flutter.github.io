---
layout: post
title: "출시 전날까지 불안했던 것들: 12명 테스트의 현실"
description: "Flutter 2048 게임의 비공개 테스트에서 점검한 튜토리얼·저장·광고·기기 조건."
date: 2026-08-18
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1753715613831-9e48eac86813?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Jakub Żerdzicki](https://unsplash.com/es/fotos/una-persona-codifica-mientras-toma-notas-QUtrcUo5-GI)

320px 폭 Android, 375×812 화면, 태블릿을 포함해 첫 실행, 강제 종료 뒤 복원, 보드 크기 전환, 되돌리기, 2,048 이후 계속하기를 점검했다. 게임 종료가 아닌 순간에 광고가 끼어들지 않는지도 확인했다.

12명 피드백은 시작점일 뿐이었다. 프로젝트의 출시 체크리스트에는 7~14일 동안 20명 이상 테스트를 권장하는 항목이 있어, 실제 공개 전에는 콘솔 요구사항과 참여 인원을 다시 맞춰야 했다.


테스트 완료 기준은 ‘크래시가 없었다’보다 구체적으로 만들었다. 최초 튜토리얼을 끝냈는지, 강제 종료 뒤 복원되는지, 광고가 오버레이를 침범하지 않는지를 반드시 확인했다.

출시 전날에는 작은 문제도 크게 보였다. 그래서 감으로 괜찮다고 넘기지 않고, 처음 설치하는 사람의 순서대로 하나씩 다시 눌러 봤다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
