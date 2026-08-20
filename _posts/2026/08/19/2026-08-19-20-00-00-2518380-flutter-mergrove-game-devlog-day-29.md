---
layout: post
title: "출시 전날, 새 기능을 꾹 참은 이유"
description: "Flutter 게임 Play 스토어 공개 전날에 버그 수정과 등록 정보만 다듬은 이유."
date: 2026-08-19
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://p0.piqsels.com/preview/60/688/850/business-office-graphic-designer-planning.jpg)

[이미지 출처: Piqsels 공개 이미지](https://www.piqsels.com/en/public-domain-photo-zkwuh)

마지막 날에는 좋은 아이디어가 가장 위험하다. 새 테마나 보상 기능은 미루고, AAB 서명, 패키지명, 개인정보처리방침 URL, 데이터 보안, 스토어 설명과 스크린샷 순서를 다시 확인했다.

Remote Config의 광고 기본값은 보수적으로 두고, 장애가 나면 광고를 끌 수 있는지 확인했다. 공개 직전의 목표는 더 많은 기능이 아니라 설치·실행·한 판 완료가 안정적인 상태였다.


공개 직전에는 변경 기록을 줄여야 원인을 추적할 수 있다. AAB와 스토어 등록정보를 고정한 뒤에는 피드백이 있어도 긴급 오류가 아닌 이상 다음 버전 후보로 분리했다.

새 기능 아이디어는 하필 마감 직전에 가장 그럴듯해 보인다. 이번에는 그 마음을 참고, 이미 만든 것이 제대로 되는지만 보기로 했다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
