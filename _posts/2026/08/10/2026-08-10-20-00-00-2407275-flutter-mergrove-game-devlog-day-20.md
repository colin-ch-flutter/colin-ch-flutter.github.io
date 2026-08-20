---
layout: post
title: "개인정보처리방침, 복붙했다가 식은땀 난 이유"
description: "Flutter 게임의 Firebase·광고 SDK와 개인정보처리방침을 맞춘 출시 준비."
date: 2026-08-10
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://p0.piqsels.com/preview/60/688/850/business-office-graphic-designer-planning.jpg)

[이미지 출처: Piqsels 공개 이미지](https://www.piqsels.com/en/public-domain-photo-zkwuh)

개인정보처리방침을 템플릿으로 끝내면 실제 SDK와 어긋나기 쉽다. 로컬 저장 데이터, Analytics·Crashlytics, Remote Config, 광고 동의 흐름을 코드 기준으로 대조했다.

‘데이터를 수집하지 않는다’고 단정하지 않고, 광고와 진단 SDK가 처리할 수 있는 항목과 선택권을 명시했다. 스토어 선언은 최종 빌드와 실제 콘솔 설정으로 다시 확인해야 한다.


정책 문구는 SDK 이름만 나열하지 않고 기능별로 연결했다. 예를 들어 로컬 저장은 게임판 복원용, Crashlytics는 오류 진단용, 광고 SDK는 동의 절차와 함께 설명했다.

정책 문서를 보며 코드에 있는 SDK를 하나씩 대조했다. 막연히 ‘개인정보는 나중에’라고 생각했던 태도가 제일 위험했다.

## 참고 자료

[Firebase for Flutter 설정](https://firebase.google.com/docs/flutter/setup) · [Firebase Remote Config 공식 안내](https://firebase.google.com/docs/remote-config/flutter/get-started)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
